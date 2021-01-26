https://issues.redhat.com/browse/OSD-6205

# Overview

When OCP installs a cluster it sets the ClusterVerison channel to the stable for that major.minor.  So if you install a cluster using an image available only in a candidate channel the in-cluster channel is still on stable.  No upgrades will be listed available until that cluster's version is in the stable channel OR the in-cluster channel is updated.

For OSD this is a challenge as customer do not have permission to manage this directly in cluster.  OCM uses telemeter to identify the channel and available upgrades for a cluster.  Therefore to boostrap this we've created this simple set of SSS to patch the channel based on the labels on ClusterDeployments (CD).

If CD indicates it is in the fast or candidate channel we use the major.minor label to pick the right SelectorSyncSet (SSS) an patch the channel in-cluster.  This gets us the initial channel corrected.  On upgrade, OCM will create an UpgradeConfig via SyncSet that contains the new target channel.  This is then set in-cluster before the upgrade is initiated.  Once upgraded, the version and channel is fed out to telemeter.

On a minor upgrade (i.e. 4.5.z to 4.6.z) it is possible that hive will reset the channel before we get updated version information.  This is OK as the upgrade is already initiated or done and the channel is only used when assessing if an upgrade can be _started_.  Once the upgrade is complete the version is sent to telemter and ultimately is reflected in the ClusterDeployment.  This in-turn will trigger the appropriate patch from SSS created in the configuration here.

NOTE that if OCP moves to a version agnostiic channel strategy that we do not need as many of these SSS and can patch just to the channel `candidate` or `fast`.

NOTE we do not need to patch to `stable` channels, as this is the default for OCP.


# Use Cases

Because there are 2 systems that are managing the in-cluster channel I wanted to walk through possible scenarios and work out if there are any conflicts.  I will use stable and candidate channels in this example but could just as easily swap out fast for candidate, the behavior is the same.

NOTE managed-upgrade-operator (MUO) is used in cluster to perform channel updates and initiate upgrade.

## Install stable, upgrade to stable

1. Install to stable channel.
2. OCM set CD to stable.
3. Hive does **not** sync any channel.
4. Customer sets upgrade schedule.
5. MUO sets channel to stable-major.minor. (noop)
6. Upgrade started.

This path works.

## Install stable, z-stream upgrade to candidate

1. Install to stable channel.
2. OCM set CD to candidate. (part of upgrade)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule.
5. MUO sets channel to candidate-major.minor. (noop)
6. Upgrade started.

This path works.

## Install stable, minor upgrade to candidate

Worst case, hive syncs old channel in the middle of things.

1. Install to stable channel.
2. OCM set CD to candidate. (part of upgrade)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule
5. MUO sets channel to candidate-major.minor**+1**
7. Hive sync candidate-major.minor to cluster channel.
8. MUO or OCP fails to find upgrade edge.
9. MUO retries upgrade.
10. MUO sets channel to candidate-major.minor**+1**
11. Upgrade started.

This path works with a retry at worst case.

## Install candidate, z-stream upgrade to candidate

1. Install to stable channel.
2. OCM set CD to candidate. (part of provisioning)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule.
5. MUO sets channel to candidate-major.minor. (noop)
6. Upgrade started.

This path works.

## Install stable, minor upgrade to candidate

Worst case, hive syncs old channel in the middle of things.

1. Install to stable channel.
2. OCM set CD to candidate. (part of upgrade)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule
5. MUO sets channel to candidate-major.minor**+1**
7. Hive sync candidate-major.minor to cluster channel.
8. MUO or OCP fails to find upgrade edge.
9. MUO retries upgrade.
10. MUO sets channel to candidate-major.minor**+1**
11. Upgrade started.

This path works with a retry at worst case.

## Install candidate, z-stream upgrade to stable

Worst case, hive syncs old channel in the middle of things.

1. Install to stable channel.
2. OCM set CD to candidate. (part of provision)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule
5. MUO sets channel to stable-major.minor
6. Hive sync candidate-major.minor to cluster channel.
7. MUO or OCP fails to find upgrade edge.
8. MUO retries upgrade.
9.  MUO sets channel to stable-major.minor
10. Upgrade started.

This path works with a retry at worst case.

## Install candidate, minor upgrade to stable

Worst case, hive syncs old channel in the middle of things.

1. Install to stable channel.
2. OCM set CD to candidate. (part of provision)
3. Hive sync candidate-major.minor to cluster channel.
4. Customer sets upgrade schedule
5. MUO sets channel to stable-major.minor**+1**
7. Hive sync candidate-major.minor to cluster channel.
8. MUO or OCP fails to find upgrade edge.
9. MUO retries upgrade.
10. MUO sets channel to stable-major.minor**+1**
11. Upgrade started.

This path works with a retry at worst case.