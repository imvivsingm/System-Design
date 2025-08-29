

# Redis Cluster Specification Explained

This document provides a simplified explanation of the Redis Cluster Specification, detailing its architecture, design principles, and operational mechanisms. Redis Cluster is a distributed implementation of Redis that offers automatic sharding, high availability, and scalability.

---

## Table of Contents

1.  [Introduction to Redis Cluster](#1-introduction-to-redis-cluster)
2.  [Core Design Goals](#2-core-design-goals)
3.  [Key Concepts & Components](#3-key-concepts--components)
    *   [Data Distribution: Hash Slots](#data-distribution-hash-slots)
    *   [Cluster Nodes & Communication](#cluster-nodes--communication)
    *   [Client Interaction & Redirection](#client-interaction--redirection)
    *   [Live Reconfiguration (Resharding)](#live-reconfiguration-resharding)
4.  [Fault Tolerance Mechanisms](#4-fault-tolerance-mechanisms)
    *   [Heartbeat & Gossip](#heartbeat--gossip)
    *   [Failure Detection](#failure-detection)
    *   [Epochs (Logical Clocks)](#epochs-logical-clocks)
    *   [Replica Election & Promotion](#replica-election--promotion)
    *   [Replica Migration](#replica-migration)
5.  [Other Features](#5-other-features)
    *   [Node Resets](#node-resets)
    *   [Removing Nodes](#removing-nodes)
    *   [Publish/Subscribe](#publishsubscribe)
6.  [In Summary](#6-in-summary)

---

## 1. Introduction to Redis Cluster

Redis Cluster enables you to run Redis across multiple nodes, overcoming the memory and CPU limitations of a single instance. It automatically shards your data, providing horizontal scalability and resilience against node failures. Unlike some other distributed systems, Redis Cluster avoids proxies and complex merging operations to maintain Redis's renowned high performance.

## 2. Core Design Goals

The design of Redis Cluster prioritizes the following:

*   **High Performance & Linear Scalability:** Achieved by direct client-to-node communication (no proxies), asynchronous replication, and avoiding costly data merge operations. Designed to scale up to 1000 nodes.
*   **Acceptable Write Safety:** Aims to preserve all writes from clients connected to the *majority* of master nodes. There are small, unavoidable windows of potential data loss during certain failure scenarios, especially for writes to minority partitions.
*   **Availability:** Can survive network partitions where the majority of masters remain reachable, provided each failed master has at least one accessible replica. Features like replica migration further enhance availability.

## 3. Key Concepts & Components

### Data Distribution: Hash Slots

*   **16384 Hash Slots:** The entire keyspace is divided into $16384$ logical slots.
*   **Master Node Ownership:** Each master node in the cluster is responsible for a unique subset of these slots.
*   **Hashing Algorithm:** Keys are mapped to slots using $ \text{HASH\_SLOT} = \text{CRC16(key)} \pmod{16384} $.
*   **Hash Tags (`{...}`):** A special mechanism to force multiple keys to reside in the same hash slot. If a key contains a substring enclosed in curly braces (e.g., `{user1000}.profile`), only the substring inside the braces (`user1000`) is used for hashing. This is essential for multi-key operations that require all keys to be on the same node.

### Cluster Nodes & Communication

*   **Unique Node ID:** Every node has a persistent, unique 160-bit ID, allowing it to retain its identity even if its IP address changes.
*   **Node Attributes:** Nodes track each other's IDs, IP addresses, ports, flags (master/replica), `configEpoch`, and the hash slots they serve.
*   **The Cluster Bus:** A dedicated TCP port (usually data port + $10000$) for high-performance, binary inter-node communication.
*   **Full Mesh Topology:** All nodes maintain direct TCP connections with every other node.
*   **Gossip Protocol:** Nodes exchange lightweight "gossip" messages to discover new nodes, propagate cluster state information, and detect failures efficiently.
*   **Node Handshake:** New nodes are introduced either manually via `CLUSTER MEET <ip> <port>` or automatically through gossip from existing trusted nodes.

### Client Interaction & Redirection

Redis Cluster employs "smart clients" that understand the cluster topology:

*   **No Proxies:** Clients connect directly to Redis nodes.
*   **`MOVED` Redirection:** If a client sends a command for a key belonging to a slot not served by the contacted node, the node responds with a `MOVED <slot> <ip>:<port>` error. The client then reissues the command to the correct node and should update its internal slot-to-node mapping.
*   **`ASK` Redirection:** Used during ongoing slot migrations (resharding). If a key is being migrated, the source node might redirect the client to the destination node using an `ASK <slot> <ip>:<port>` error. The client must prefix the redirected command with `ASKING` to signal it's a temporary redirection. This allows new writes for the migrating slot to go to the new master, while existing keys are still handled by the old master until fully moved.

### Live Reconfiguration (Resharding)

Redis Cluster supports adding, removing, and rebalancing nodes dynamically without downtime:

*   **Slot Migration:** The core operation involves moving all keys within a hash slot from a source master to a destination master.
*   **Commands:** `CLUSTER ADDSLOTS`, `CLUSTER DELSLOTS`, `CLUSTER SETSLOT <slot> MIGRATING <node-id>`, `CLUSTER SETSLOT <slot> IMPORTING <node-id>`.
*   **Atomic Key Transfers:** The `MIGRATE` command efficiently and atomically moves individual keys between instances.

## 4. Fault Tolerance Mechanisms

### Heartbeat & Gossip

Nodes continuously exchange `PING` and `PONG` packets (heartbeats) over the cluster bus. These packets carry vital configuration information and are used for:
*   Discovering new nodes.
*   Propagating cluster state.
*   Detecting unreachable nodes.

### Failure Detection

*   **`PFAIL` (Possible Failure):** A node marks another node as `PFAIL` if it's unreachable for longer than `NODE_TIMEOUT`. This is a local observation.
*   **`FAIL` (Confirmed Failure):** A `PFAIL` condition escalates to `FAIL` if a *majority of master nodes* also report the same node as `PFAIL` or `FAIL` within a specific timeframe. Once confirmed, a `FAIL` message is broadcast to ensure all nodes acknowledge the failure.

### Epochs (Logical Clocks)

Epochs are 64-bit unsigned numbers used to provide incremental versioning for cluster events and resolve conflicts:

*   **`currentEpoch`:** A cluster-wide logical clock. Nodes update their `currentEpoch` if they receive a message with a higher epoch value. All nodes eventually converge to the highest `currentEpoch`.
*   **`configEpoch`:** A unique `configEpoch` is assigned to each master node for the hash slots it serves. A higher `configEpoch` signifies a more recent and authoritative configuration. This is crucial during failovers and resharding to ensure all nodes agree on the current master for a set of slots.

### Replica Election & Promotion

When a master node is detected as `FAIL`:

1.  **Election Initiation:** One of its replicas (the most up-to-date one, determined by "replica rank") initiates an election.
2.  **Vote Request:** The replica increments its `currentEpoch` and broadcasts `FAILOVER_AUTH_REQUEST` messages to all master nodes.
3.  **Majority Vote:** If the replica receives votes from a majority of master nodes, it wins the election.
4.  **Promotion:** The winning replica obtains a new, unique, and higher `configEpoch`, promotes itself to master, and starts serving the failed master's slots. Other nodes update their configuration to reflect this change.

### Replica Migration

This feature enhances availability by dynamically rebalancing replicas:

*   If a master node loses all its replicas (becomes "orphaned"), a replica from another master (that has a surplus of replicas) can "migrate" to become a replica of the orphaned master.
*   This ensures that every master ideally has at least one replica, making the cluster more resilient to subsequent failures.
*   A `cluster-migration-barrier` setting prevents replicas from migrating if their master would be left with too few replicas.

## 5. Other Features

### Node Resets

The `CLUSTER RESET` command (soft or hard) allows an administrator to reset a node's state, clearing its slots, removing knowledge of other nodes, and potentially changing its Node ID and epochs. This is useful for repurposing nodes.

### Removing Nodes

To remove a node, its data must first be resharded away (if it's a master). Then, the `CLUSTER FORGET <node-id>` command is used on other nodes to remove its entry from their tables. A 60-second ban prevents the node from being re-added immediately via gossip.

### Publish/Subscribe

*   **Traditional Pub/Sub:** Clients can subscribe/publish to any node, and messages are broadcast cluster-wide.
*   **Sharded Pub/Sub (Redis 7.0+):** Channel names are hashed to specific slots. Publishers send messages to the master of that slot (or its replicas). This improves scalability for Pub/Sub by directing traffic to specific shards.

## 6. In Summary

The Redis Cluster Specification outlines a robust, high-performance distributed system designed to scale Redis horizontally. It achieves fault tolerance through a combination of smart clients, gossip-based failure detection, epoch-driven conflict resolution, and automatic replica election/migration. This pragmatic approach allows Redis to maintain its characteristic speed and simplicity while operating as a resilient, distributed data store.

---
