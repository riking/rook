---
title: Data Pool Configuration
weight: 2000
indent: true
---

# Ceph Data Pool Configuration

Each storage pool and derived service (Block Pools, Object Store, Shared
Filesystem, NFS, ...) that you create will need the dataPool settings configured
for proper durability of your data.

There are two primary choices: Replicated and Erasure Coding.

## Replicated

Replicated storage is the simplest option and provides the best performance and
lowest node count.

```yaml
dataPool:
  replicated:
    size: 3
```

Common settings are 2x and 3x replication. 2x is not recommended except for
data that can be easily recreated, as a single unexpected failure during
maintenance will result in the cluster being unable to serve reads for data on
the two affected hosts.

If your cluster only has 3 nodes, then 3x replication is likely the best choice
despite the 200% space penalty: the cluster will continue to be writeable during
maintenance, unlike 2:1 erasure coding.

## Erasure Coded

Erasure coded pools require the OSDs to use `bluestore` for the configured `storeType`.

[Erasure coding](http://docs.ceph.com/docs/master/rados/operations/erasure-code/) allows you to keep your data safe while reducing the storage overhead. Instead of creating multiple replicas of the data,
erasure coding divides the original data into chunks of equal size, then generates extra chunks of that same size for redundancy.

For example, if you have an object of size 2MB, the simplest erasure coding with two data chunks would divide the object into two chunks of size 1MB each (data chunks). One more chunk (coding chunk) of size 1MB will be generated. In total, 3MB will be stored in the cluster. The object will be able to suffer the loss of any one of the chunks and still be able to reconstruct the original object.

The erasure coding profiles available to you depend on the number of nodes in
your cluster running OSDs, how much of a storage penalty you want to take, and
how many failures need to be tolerated while keeping the data accessible.

## Choosing a configuration

In the below table, `Nx` refers to a replicated configuration with N replicas,
and `k:m` refers to an erasure coding configuration with `k` data chunks and `m`
coding chunks.

| Configuration | Storage penalty | Losses tolerated | Nodes required (for
HEALTH\_OK) |
| ------------- | --------------- | ---------------- | -------------- |
| 2x            | 100%            | 1                | 2 (3)          |
| 3x            | 200%            | 2                | 3 (4)          |
| 2:1           | 50%             | 1                | 3 (4)          |
| 2:2           | 100%            | 2                | 4 (6)          |
| **3:2**       | 66%             | 2                | 5 (7)          |
| 4:2           | 50%             | 2                | 6 (8)          |
| **4:3**       | 75%             | 3                | 7 (10)         |
| 10:4          | 40%             | 4                | 14 (18)        |
| 16:4          | 25%             | 4                | 20 (24)        |

Take into account node time to repair when evaluating configurations with higher
node requirements. In general, as the number of nodes required by a
configuration go up, the number of losses that need to be tolerated to provide a
given data durability also increases. The number of nodes that should be
available for data rebalancing upon a failure also increases.
