---
rskip: 68
title: Federation Notification System
description: 
status: Draft
purpose: Sec
author: JIO <jorlicki@iovlabs.org>, SDL (@sergiodemianlerner)
layer: Net
complexity: 2
created: 2018-05-05
---


# Federation Notification System

|RSKIP          |68           |
| :------------ |:-------------|
|**Title**      |Federation Notification System |
|**Created**    |05-MAY-18 |
|**Author**     |SDL, JIO, DAM |
|**Purpose**    |Sec |
|**Layer**      |Net |
|**Complexity** |2 |
|**Status**     |Draft* |

# Abstract

In this RSKIP we propose a mechanism to allow members of the Federation periodically notify hashes of blocks at different heights from their best chains. This mechanism is intended to minimize the impact of potential attacks from Bitcoin miners to the RSK network. These notifications are merely informative the final decision on how to act upon a discrepancy between the Federation nodes and the local node receiving the notifications relies on the local node.

# Motivation

Due to the relative small hashing power the RSK network has in comparison with Bitcoin there is a chance of potential attacks from Bitcoin merge-miners to the RSK network. 

Possible attacks include:

* Fork attacks: attempt to force RSK nodes to accept (or work in the case of miners) on a fork of the current best chain for a long period of time.
* Eclipse attacks: attempt to isolate RSK nodes from the network.

To minimize the impact of the first type of attack there should be a secondary source from where nodes can check the status of the best chain. We propose the Federation nodes as such source of information. They communicate its view of the best chain through notifications. These notifications would be broadcasted periodically to the entire network. The notifications should include hashes of blocks at different heights from the best chains of Federation members.

The information received in the Federation notifications will allow nodes to verify if their best chains diverge from the one reported by the Federation members and chose what actions (if any) to take in such case.

For the second type of attack the absence of Federation messages could be used as evidence to the nodes that they might be victims of a potential Eclipse attack.

Additionally federation anomalous behavior can be detected with this mechanism, in particular nodes will be able to detect when federation members are frozen, meaning, no new blocks are processed by the federation members. This allows us to distinguish between a potential eclipse attack and federators not processing new blocks, for instances, as a result of a bug in federators code or a network problem. 

# Specification

We propose the introduction of a new message type, the FederationNotification. Members of the Federation will periodically broadcast these notifications to the network. The notifications will carry a list a confirmations from the federation best chain. A confirmation is just a pair of block number and block hash. The first confirmation in the list will correspond to a block near the tip of the best chain (e.g. ten blocks bellow the tip), additional confirmations in the list will correspond to blocks deeper in best chain (e.g. 20 blocks for the second confirmation, 40 blocks for the third confirmation and so on). Confirmations are intended to allow nodes to verify if they are following the Federation's best chain or if they diverge from it. Nodes would be able to choose which level of verification to perform, meaning which of the provided confirmations to use.

The following list shows all the fields included in the FederationNotification:

* source: Address of the Federation member that originated the notification.
* confirmations: a list of pairs (best chain blockNumber and blockHash), for each verification level.
* timeToLive: How long this notification will be valid.
* timestamp: When this notification was generated.
* signature: A signature of all the above fields generated using the private key of the Federation member that originated the notification.

It is worth to mention that the delivery of these notifications should not flood the network with messages. One approach to fulfill this requirement could be: when a member of the federation receives a new block and a configurable timer expires a new FederationNotification will be broadcasted to the network.

To detect frozen federation members the notifications should also be sent when no block is received by the federation members and a configurable timer expires.

Nodes should also count with a mechanism to prevent an attacker to attempt flooding the network with Federation Notifications. 

Upon receiving a Federation notification, nodes should check if it is not an expired notification and if it wasn't received previously. Then, nodes should verify the notification origin, generate any alerts if needed and propagate the message to its peers. Nodes should also update the last time they received a notification from the Federation. This value will be used to detect potential Eclipse attacks.

The following fragment of Java code reflects the behavior described: 

```java
/**
 * Validates the received FederationNotification, if valid forwards the
 * notification to the activePeers collection and generates ForkAttackAlerts if
 * a fork attack is detected.
 *
 * @throws ConfigurationException
 */
@Override
public FederationNotificationProcessingResult processFederationNotification(
        @Nonnull final Collection<Channel> activePeers, @Nonnull final FederationNotification notification)
        throws ConfigurationException {
    long timeBetweenNotifications = Duration.between(lastNotificationReceivedTime, Instant.now()).getSeconds();

    // Avoid flood attacks
    if (lastNotificationReceivedTime != null
            && timeBetweenNotifications < config.maxSecondsBetweenNotifications()) {
        logger.warn("Federation notification received too fast. Skipping it to avoid flood attacks.");
        return FederationNotificationProcessingResult.NOTIFICATION_RECEIVED_TOO_FAST;
    }

    // Reject expired notifications
    if (notification.isExpired()) {
        logger.warn("Federation notification has expired.");
        return FederationNotificationProcessingResult.NOTIFICATION_EXPIRED;
    }

    // Reject already received notifications
    if (receivedFederationNotifications.containsNotification(notification)) {
        logger.info("Federation notification already processed.");
        return FederationNotificationProcessingResult.NOTIFICATION_ALREADY_PROCESSED;
    }

    // Only accept notifications from valid federation members
    if (!verifyFederationNotificationSignature(notification)) {
        logger.warn("Federation notification signature does not verify.");
        return FederationNotificationProcessingResult.NOTIFICATION_SIGNATURE_DOES_NOT_VERIFY;
    }

    // Cache notification to keep track of already received notifications
    this.receivedFederationNotifications.addNotification(notification);

    // Compare confirmations contained in the notification with local best
    // chain to generate alerts (if needed)
    this.generateAlertIfNeeded(notification);

    // Propagate federation notification to peers
    broadcastFederationNotification(activePeers, notification);

    // Update timestamp of last notification received to later check if
    // communications with federation are still alive
    this.lastNotificationReceivedTime = Instant.now();

    logger.debug("Federation notification processed successfully.");
    return FederationNotificationProcessingResult.NOTIFICATION_PROCESSED_SUCCESSFULLY;
}
```

To detect potential Eclipse attacks we just need to track the delta in time between Federation Notifications. If this delta exceeds a configurable threshold then we assume the node might be under an Eclipse attack. The following fragment of Java code shows how this behavior can be implemented:

```java
/**
 * Checks the time of the last FederationNotification received does not exceed a
 * configurable delta. If so, a FederationEclipsedAlert is generated and the panic
 * status of the processor is set to FEDERATION_ECLIPSED. This alert indicates that
 * that either the node was isolated from the federation by an attacker or the
 * federation nodes are offline.
 */
@Override
public void checkIfFederationWasEclipsed() {
    // Maybe someone is eclipsing the Federation for this node or the federation nodes are offline.
    Duration duration = Duration.between(lastNotificationReceivedTime, Instant.now());
    int maxSilenceSecs = config.getFederationMaxSilenceTimeSecs();
    if (config.federationNotificationsEnabled() && duration != null && duration.getSeconds() > maxSilenceSecs) {

        FederationAlert alert = new FederationEclipsedAlert(duration.getSeconds());
        addFederationAlert(alert);

        if (config.shouldFederationNotificationsTriggerPanic()) {
            setPanicStatus(PanicStatus.FederationEclipsedPanic(getBestBlockNumber()));
            logger.warn(
                    "Federation alert generated. Panic status is {}. No Federation Notifications received for {} seconds",
                    getPanicStatus(), duration.getSeconds());
        }
    } else {
        if (config.shouldFederationNotificationsTriggerPanic()) {
            setPanicStatus(PanicStatus.NoPanic(getBestBlockNumber()));
        }
    }
}
``` 

To detect frozen federation members nodes can use the isFederationFrozen() method of the FederationNotification:

```java
if (notification.isFederationFrozen()) {
    FederationAlert alert = new FederationFrozenAlert(notification.getSource(), c.getBlockHash(), c.getBlockNumber());
    addFederationAlert(alert);

    long panicSinceBlock = getBestBlockNumber();
    setPanicStatus(PanicStatus.FederationFrozenPanic(panicSinceBlock));

    logger.warn("Federation alert generated. Panic status is {}", getPanicStatus());

    return;
}
```

Generated alerts will remain active until the processing a new Federation Notification clears the alert reason. In the case of a potential Eclipse attack this means receiving a non-expired Federaton Notification. In the case of a fork attack this means discarding the fork and receiving a Federation Notification which confirmations that match the current best chain of the node. In the case of a frozen federation member this means receiving a Federation Notification whose isFederationFrozen() method returns false.

When triggering an alert the node will also transition to a panic status. The panic status contains the reason of the panic and the block number when the panic started. Panic status remains active following the same rules as Alerts. So far a node in a panic status will not stop working, but additional logic can be added to the node to forbid certain actions when in panic state.

Both, current panic status and a list with the latest alerts can be queried using the Javascript API.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

