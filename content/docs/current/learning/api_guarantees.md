---
title: KV API guarantees
weight: 2750
description: KV API guarantees made by etcd
---

etcd is a consistent and durable key value store with [mini-transaction][txn] support. The key value store is exposed through the KV APIs. etcd tries to ensure the strongest consistency and durability guarantees for a distributed system. This specification enumerates the KV API guarantees made by etcd.

### APIs to consider

* Read APIs
    * range
    * watch
* Write APIs
    * put
    * delete
* Combination (read-modify-write) APIs
    * txn
* Lease APIs
    * grant
    * revoke
    * put (attaching a lease to a key)

### etcd specific definitions

#### Operation completed

An etcd operation is considered complete when it is committed through consensus, and therefore “executed” -- permanently stored -- by the etcd storage engine. The client knows an operation is completed when it receives a response from the etcd server. Note that the client may be uncertain about the status of an operation if it times out, or there is a network disruption between the client and the etcd member. etcd may also abort operations when there is a leader election. etcd does not send `abort` responses to  clients’ outstanding requests in this event.

#### Revision

An etcd operation that modifies the key value store is assigned a single increasing revision. A transaction operation might modify the key value store multiple times, but only one revision is assigned. The revision attribute of a key value pair that was modified by the operation has the same value as the revision of the operation. The revision can be used as a logical clock for key value store. A key value pair that has a larger revision is modified after a key value pair with a smaller revision. Two key value pairs that have the same revision are modified by an operation "concurrently".

### Guarantees provided

#### Atomicity

All API requests are atomic; an operation either completes entirely or not at all. For watch requests, all events generated by one operation will be in one watch response. Watch never observes partial events for a single operation.

#### Durability

Any completed operations are durable. All accessible data is also durable data. A read will never return data that has not been made durable.

#### Isolation level and consistency of replicas

etcd ensures [strict serializability][strict_serializability], which is the strongest isolation guarantee of distributed transactional database systems. Read operations will never observe any intermediate data.

etcd ensures [linearizability][linearizability] as consistency of replicas basically. As described below, exceptions are watch operations and read operations which explicitly specifies serializable option.

From the perspective of client, linearizability provides useful properties which make reasoning easily. This is a clean description quoted from [the original paper][linearizability]: `Linearizability provides the illusion that each operation applied by concurrent processes takes effect instantaneously at some point between its invocation and its response.`

For example, consider a client completing a write at time point 1 (*t1*). A client issuing a read at *t2* (for *t2* > *t1*) should receive a value at least as recent as the previous write, completed at *t1*. However, the read might actually complete only by *t3*. Linearizability guarantees the read returns the most current value. Without linearizability guarantee, the returned value, current at *t2* when the read began, might be "stale" by *t3* because a concurrent write might happen between *t2* and *t3*.

etcd does not ensure linearizability for watch operations. Users are expected to verify the revision of watch responses to ensure correct ordering.

etcd ensures linearizability for all other operations by default. Linearizability comes with a cost, however, because linearized requests must go through the Raft consensus process. To obtain lower latencies and higher throughput for read requests, clients can configure a request’s consistency mode to `serializable`, which may access stale data with respect to quorum, but removes the performance penalty of linearized accesses' reliance on live consensus.


### Granting, attaching and revoking leases

etcd provides [a lease mechanism][lease]. The primary use case of a lease is implementing distributed coordination mechanisms like distributed locks. The lease mechanism itself is simple: a lease can be created with the grant API, attached to a key with the put API, revoked with the revoke API, and will be expired by the wall clock time to live (TTL). However, users need to be aware about [the important properties of the APIs and usage][why] for implementing correct distributed coordination mechanisms.

[txn]: api.md#transactions
[linearizability]: https://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf
[strict_serializability]: http://jepsen.io/consistency/models/strict-serializable
[serializable_isolation]: https://en.wikipedia.org/wiki/Isolation_(database_systems)#Serializable
[Linearizability]: #Linearizability
[lease]: https://web.stanford.edu/class/cs240/readings/89-leases.pdf
[why]: why.md#Notes