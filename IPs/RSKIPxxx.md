# Tokenization of Ecosystem Fund in REMASC 

|RSKIP          |XXX           |
| :------------ |:-------------|
|**Title**      |Tokenization of Ecosystem Fund in REMASC |
|**Created**    |5-MAY-2021 |
|**Author**     |SDL |
|**Purpose**    |Fairness |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |
|**discussions-to**     ||

# **Abstract**

This RSKIP proposes the tokenization of 19% of REMASC payments currently assigned to IOV Labs.


# **Motivation**

The REMASC smart-contract distributes transaction fees among miners (80%), IOV Labs  (19%) and the pegnatories (1%). A 19% share was allocated to IOV Labs to fund RSK's past and future development. So far that distribution has been done through IOV Labs but the RSK platform is mature enough to start decentralizing this process. This will bring great benefits to RSK by expanding its network of collaborators while making the RSK ecosystem stronger as a whole.

# **Specification**

The new token name should be defined by the RSK community. In this proposal it is temporarily called Ging in honor of the first RSK release name (Ginger). There will be a single token generation event, where one S will be created (S=1_000_000). Ging tokens won't be divisible.

The Ging will be managed by one  special ERC20 contract caled GingToken, and one standard ERC20 token called BtcCollecorToken. Both are implemented in Solidity. 

## GingToken

Each address holding Ging in GingToken collects bitcoins for the same address in the BtcCollecorToken contract. If the user wants to control the bitcoins earned from a different contract, it can perform an infinite approve to a different address in BtcCollecorToken.

The basic definition is the following:

```Solidity
pragma solidity ^0.4.18;

contract GingToken {
    string public name     = "Ging";
    string public symbol   = "gng";
    uint8  public decimals = 0;
    uint   S = 1000000;
.....
}
```

For every address the GingToken maintains two additional fields:

* satFeesPaid (uint64): total fees paid to this address.
* satFeesReceived (uint64): this is a checkpoint indicating the amount of fees globally received the last time a fee collection was performed for this address.

These fields can be stored in a struct together with the Ging balance or in  separate maps (TBD). Now we'll assume each field is stored in an independent map:

    mapping (address => uint64)  public  satFeesPaid;
    mapping (address => uint64)  public  satFeesReceived;

The GingToken maintains an monolithically increasing accumulators totalSatFeesPaid. All bitcoin-denominated state variables (satFeesReceived, satBalance, totalSatFeesPaid) are represented internally in satoshis (not wei) to save space.

```Solidity
uint64 totalSatFeesPaid;
```

This method must be implemented:

```Solidity
function getTotalSatFeesReceived() returns (uint64) private {
 return (this.balance+totalSatFeesPaid);
}
```

Every time a Ging transfer method is called, but before the transfer occurs, GingToken will call the public method collectFees(x), where x is the source account. Here it is defined:

```Solidity
function collectFees(address x) {
    uint totalReceived = getTotalSatFeesReceived();
    if (totalReceived>satFeesReceived[x]) {
      // The followin computation is similar to getPendingBtcFees(x), 
      // but saves computing getTotalSatFeesReceived() twice, and uses sat instead of wei
      uint transferAmount =(totalReceived-satFeesReceived[x])*balance[x]/S; 
      satFeesReceived[x]  =totalReceived;
      satFeesPaid[x]     +=transferAmount;
      totalSatFeesPaid   +=transferAmount;
      btccollectorToken.transfer(x,transferAmount);
    }
}

function getPendingBtcFees(address x) return (uint) public {
    return (getTotalSatFeesReceived()-satFeesReceived[x])*balance[x]*satToWei/S;
}
```

The suggested implementation for transfers is the following:

```Solidity
   function transfer(address dst, uint wad) public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }

    function transferFrom(address src, address dst, uint wad)
        public
        returns (bool)
    {
        collectFees(src);
        // standard transferFrom() code goes here...
    }
}
```

Anyone can call collectFees(x) at any time. 

Other projectors are the following:

```Solidity
function getTotalBtcFeesPaid() returns (uint) {
  return totalSatFeesPaid*satToWei;
}

function getBtcFeesPaid(address x) returns (uint) {
  return satFeesPaid[x]*satToWei;
}

function getBtcFeesReceived(address x) returns (uint) {
  return satFeesReceived[x]*satToWei;
}
```

These methods return the value in wei (to be compatible with RSK). The use of the satoshi unit  is internal to the GingToken and the user always interact with the contract in wei units.

To query the amount of bitcoins pending to be paid, this method is defined:
```Solidity
function getTotalSatFeesToBePaid() private {
 return getTotalSatFeesReceived()-totalSatFeesPaid;
}
```

The bitcoins earned are moved to the contract BtcCollecorToken, which is  a wrapped-btc ERC20 contract, but with the embedded capability to collect fees on demand, while reflecting the balance of an address as if all pending fees had been received.

The fees that are pending to collect can be retrieved by the method getPendingBtcFees(x). 

## BtcCollecorToken

When the BtcCollectorContract queries the balance of an address x with balanceOf(x), the BtcCollectorContract will return balance[x] plus gingToken.getPendingBtcFees(x). When the BtcCollectorContract performs a transfer operation with address x as source, it will first perform gingToken.collectFees(x). 

The following is the suggested contract code:

```Solidity
pragma solidity ^0.4.18;

contract BtcCollecorToken {
    string public name     = "BtcCollecor";
    string public symbol   = "cBTC";
    uint8  public decimals = 18;

    event  Approval(address indexed src, address indexed guy, uint wad);
    event  Transfer(address indexed src, address indexed dst, uint wad);
    event  Deposit(address indexed dst, uint wad);
    event  Withdrawal(address indexed src, uint wad);

    mapping (address => uint)                       public  balanceOf;
    mapping (address => mapping (address => uint))  public  allowance;
    
    GingToken gingToken; // write a suitable constructor to set this

    function balanceOf(address _owner) public view returns (uint256 balance) {
      return   balanceOf[_owner]+gingToken.getPendingBtcFees(_owner);
    }
    
    function() public payable {
        deposit();
    }
    
    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        Deposit(msg.sender, msg.value);
    }
    function withdraw(uint wad) public {
        gingToken.collectFees(msg.sender);
        require(balanceOf[msg.sender] >= wad);
        balanceOf[msg.sender] -= wad;
        msg.sender.transfer(wad);
        Withdrawal(msg.sender, wad);
    }

    function totalSupply() public view returns (uint) {
        return this.balance+gingToken.getTotalSatFeesToBePaid();
    }

    function approve(address guy, uint wad) public returns (bool) {
        allowance[msg.sender][guy] = wad;
        Approval(msg.sender, guy, wad);
        return true;
    }

    function transfer(address dst, uint wad) public returns (bool) {
        return transferFrom(msg.sender, dst, wad);
    }

    function transferFrom(address src, address dst, uint wad)
        public
        returns (bool)
    {
        gingToken.collectFees(msg.sender);
        require(balanceOf[src] >= wad);

        if (src != msg.sender && allowance[src][msg.sender] != uint(-1)) {
            require(allowance[src][msg.sender] >= wad);
            allowance[src][msg.sender] -= wad;
        }

        balanceOf[src] -= wad;
        balanceOf[dst] += wad;

        Transfer(src, dst, wad);

        return true;
    }
}
```

Note that if the user wants to re-invest the bitcoins received in Ging, it must call withdraw bitcoins from BtcCollecorToken, swap the bitcoins for Gings in an exchage, and transfer the Gings back to its account in the GingToken.

At the REMASC is be updated so that the IOV share is transferred to the GingToken contract after the hard-fork activation.

# Rationale


# Backwards Compatibility

This change is a hard-fork and therefore all full nodes must be updated. SPV light-clients and Block explorers that are always in sync do not need to be updated. 

# Test Cases

TBD

## Security Considerations

TBD


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
