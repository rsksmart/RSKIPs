---
rskip: 36
title: Transaction Encapsulation
description: 
status: Draft
purpose: Sca
author: SDL (@sergiodemianlerner)
layer: Core
complexity: 2
created: 2017-02-02
---

# Transaction Encapsulation 

|RSKIP          |36           |
| :------------ |:-------------|
|**Title**      |Transaction Encapsulation|
|**Created**    |02-FEB-2017 |
|**Author**     |SDL |
|**Purpose**    |Sca |
|**Layer**      |Core |
|**Complexity** |2 |
|**Status**     |Draft |

# **Abstract**

To achieve true financial inclusion within the next 10 years we need to allow users to hold their funds in a local/fiat cryptocurrency. Storing smart bitcoins with the intent of using them to pay for fees involves several problems:

* Bitcoin involves a price volatility risk

* Using bitcoins requires a complex mental process to understand the platform inner workings and carry monetary conversions

* Monetary conversions require time, both from the user and internally.

* The ownership of bitcoins may not be viewed positively by the masses.

Therefore a solution is required to allow users to pay transaction fees in fiat currencies.

Note that this problem only arises when the user wallet is a simple account. When the user wallet is a smart contract, then the smart contract can (and should!) be made such that the payment is encapsulated in a message (not an external transaction), and the message is signed. A smart contract wallet should never authenticate the user using msg.sender or msg.origin. When the user wallet is a smart contract, the user can send the signed payload to a third party where the user has a minimum fiat account balance, and this third party would encapsulate the payload and send it to the wallet smart contract. The cost of tunneling every payment through a smart wallet is 700 gas for the additional CALL, plus the execution of code that deserializes and interprets the payload, which should be another 700 gas, totalling 1400 gas, which is less than 10% of the external transaction cost. This RSKIP defines the EXECUTE opcode, to allow account-based transfers to be executed by third parties which pay for the transaction fees.


# **Specification**

See discussion [here](https://github.com/rsksmart/RSKIPs/issues/82)

## A new opcode EXECUTE is added.

Arguments: <transactionData> <maxgas>

Returns: 

This method works like a CALL, but no arguments can be sent or received.

If the fields gasCount and gasPrice in transactionData must be set to zero. 

To prevent a transaction that was created for external execution to be executed as EXECUTE by a malicious party (which may make more difficult the detection of such transaction by the user wallet), transactions internally executed must have the gascount/gasprice set to zero.

The cost of EXECUTE is 4000. If the encapsulated transaction transfers value, an additional cost of 2300 is added. 

 

**Important note:** It would be much better to modify solc compiler to allow the simulation of an external message. This is done by allowing the input arguments to be loaded from another source. Each contract would have an implicit method: _call(byte[] ABIpayload)

# Appendix A

pragma solidity ^0.4.6;

contract Token {

    /// total amount of tokens

    uint256 public totalSupply;

    function balanceOf(address _owner) constant returns (uint256 balance);

    function transfer(address _to, uint256 _value) returns (bool success);

    function transferFrom(address _from, address _to, uint256 _value) returns (bool success);

    function approve(address _spender, uint256 _value) returns (bool success);

    function allowance(address _owner, address _spender) constant returns (uint256 remaining);

    event Transfer(address indexed _from, address indexed _to, uint256 _value);

    event Approval(address indexed _owner, address indexed _spender, uint256 _value);

}

contract StandardToken is Token {

    address public creator;
    

    function doNothing() {        
    }

    

    function transfer(address _to, uint256 _value) returns (bool success) {

        //if (balances[msg.sender] >= _value && balances[_to] + _value > balances[_to]) {
        if (balances[msg.sender] >= _value && _value > 0) {
            balances[msg.sender] -= _value;
            balances[_to] += _value;
            Transfer(msg.sender, _to, _value);
            return true;
        } else { return false; }
    }

    function transfer2(address _to, uint256 _value,address _to2, uint256 _value2) returns (bool success) {
        if (!transfer(_to,_value))
            return false;
        return transfer(_to2,_value2);
    }

    function transferFrom(address _from, address _to, uint256 _value) returns (bool success) {

        if (balances[_from] >= _value && allowed[_from][msg.sender] >= _value && _value > 0) {
            balances[_to] += _value;
            balances[_from] -= _value;
            allowed[_from][msg.sender] -= _value;
            Transfer(_from, _to, _value);
            return true;
        } else { return false; }
    }

    function balanceOf(address _owner) constant returns (uint256 balance) {
        return balances[_owner];
    }

    function approve(address _spender, uint256 _value) returns (bool success) {
        allowed[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
        return true;
    }

    function allowance(address _owner, address _spender) constant returns (uint256 remaining) {
      return allowed[_owner][_spender];
    }   

    function StandardToken() {
        balances[msg.sender] = 1000;
        creator = msg.sender;
        totalSupply = 1000;                        // Update total supply
    }

    mapping (address => uint256) balances;
    mapping (address => mapping (address => uint256)) allowed;
}


# **Copyright**

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).