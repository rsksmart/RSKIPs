RSKIP: 
  Title: Federation Notification System
  Status: Draft
  Type: Net
  Authors: Sergio Lerner <sergio@rsk.co>, Jose Orlicki <jorlicki@rsk.co>, Diego Masini <dmasini@rsk.co>
  Created: 2018-05-04

# Abstract

In this RSKIP we propose a mechanism that allows members of the Federation to periodically notify hashes of blocks at different heights from their best chains. This mechanism is intended to be used to minimize the impact of potential attacks from Bitcoin miners to the RSK network. These notifications are merely informative, the final decision on how to act upon a discrepancy between the Federation nodes and the local node receiving the notifications relies on the local node.

# Motivation

Due to the relative small hashing power the RSK network has in comparison with Bitcoin there is a chance of potential attacks from Bitcoin merge-miners to the RSK network. 

Possible attacks include:

* Fork attacks: attempt to force RSK nodes to accept (or work in the case of miners) on a fork of the current best chain for a long period of time.
* Eclipse attacks: attempt to isolate RSK nodes from the network.

To minimize the impact of the first type of attack there should be a secondary source from where nodes can check the status of the best chain. We propose the Federation nodes as such source of information. They communicate its view of the best chain through notifications. These notifications would be broadcasted periodically to the entire network. The notifications should include hashes of blocks at different heights from the best chains of Federation members.

The information received in the Federation notifications will allow nodes to verify if their best chains diverge from the one reported by the Federation members and chose what actions (if any) to take in such case.

For the second type of attack the absence of Federation messages could be used as evidence to the nodes that they might be victims of an Eclipse attack.

# Specification

We propose the introduction of a new message type, the FederationNotification. Members of the Federation will periodically broadcast these notifications to the network. These notifications will carry information from a list of blocks from the current best chain (block number and block hash for each of these blocks). The first block should be near the tip of the best chain (e.g. ten blocks bellow the tip) and its information is intended for nodes to verify if they are closer to the Federation's best chain. The last block should be located far from the tip of the best chain (e.g. one hundred blocks below the tip) and its information is intended for the nodes to detect if they deviated drastically from the Federation's best chain. Other block heights in between, for example 25. Nodes would be able to choose which level of verification to perform, meaning which of the provided block numbers and block hashes to use.

The following list shows all the fields included in the FederationNotification:

* source: Address of the Federation member that originated the notification.
* confirmations: a list of pairs (best chain blockNumber and blockHash), for each verification level.
* timeToLive: How long this notification will be valid.
* timestamp: When this notification was generated.
* signature: A signature of all the above fields generated using the private key of the Federation member that originated the notification.

It is worth to mention that the delivery of these notifications should not flood the network with messages. One approach to fulfill this requirement could be: when a member of the federation receives a new block and a configurable timer expires a new FederationNotification will be broadcasted to the network.

Nodes should also count with a mechanism to prevent an attacker to attempt flooding the network with Federation Notifications. 

Upon receiving a Federation notification, nodes should check if it is not an expired notification and if it wasn't received previously. Then, nodes should verify the notification origin, generate any alerts if needed and propagate the message to its peers. Nodes should also update the last time they received a notification from the Federation. This value will be used to detect potential Eclipse attacks.

The following fragment of Java code reflects the behavior described: 

```java
public FederationNotificationProcessingResult processFederationNotification(@Nonnull final Collection<Channel> activePeers, @Nonnull final FederationNotification notification) {
    long elapsedTimeBetweenNotifications = ChronoUnit.SECONDS.between(lastNotificationReceivedTime, Instant.now());

    // Avoid flood attacks
    if (lastNotificationReceivedTime != null && elapsedTimeBetweenNotifications < config.getFederationDeltaTimeForNotificationsSecs()) {
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

To detect Eclipse attacks we just need to track the delta in time between Federation Notifications. If this delta exceeds a configurable threshold then we assume the node might be under an Eclipse attack. The following fragment of Java code shows how this behavior can be implemented:

```java
public void checkIfUnderEclipseAttack() {
    Duration duration = Duration.between(lastNotificationReceivedTime, Instant.now());
    int maxSilenceSecs = config.getFederationMaxSilenceTimeSecs();
    if (config.federationNotificationsEnabled() && duration != null && duration.getSeconds() > maxSilenceSecs) {

        FederationAlert alert = new EclipseAttackAlert(duration.getSeconds());
        addFederationAlert(alert);

        if (config.shouldFederationNotificationsTriggerPanic()) {
            setPanicStatus(PanicStatus.UnderEclipseAttackPanic(getBestBlock().getNumber()));
            logger.warn("Federation alert generated. Panic status is {}. No Federation Notifications received for {} seconds", getPanicStatus(), duration.getSeconds());
        }
    } else {
        if (config.shouldFederationNotificationsTriggerPanic()) {
            setPanicStatus(PanicStatus.NoPanic(getBestBlock().getNumber()));
        }
    }
}
``` 

Generated alerts will remain until a new Federation Notification triggers the clearing of the alert reason. In the case of an Eclipse attack this means receiving a non-expired Federaton Notification. In the case of a fork attack this means discarding the fork and receiving a Federation Notification which confirmations that match the current best chain of the node. 

When triggering an alert the node will also enter a panic status. Panic status contains the reason of the panic and the block number when the panic started. Panic status remains active following the same rules as Alerts.

Both, current panic status and a list with the latest alerts can be queried using the Javascript API.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

