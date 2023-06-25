# Distributed Key-Value Store #

**Run Command:**

To run a server independently:

```java -Xmx512m -jar A12.jar <ipAddress> <portNumber>```

To launch multiple sub-processes of the above command, there is a python script:

```python3 launch.py <numInstances>```

## Brief Description:

### Consistent Hashing:
We are using guava's consistent hash api to map hashes into node buckets. The hash function we are using for keys is MD5.
If a request is routed to another member node, that node is responsible for sending a reply to the original client
(or test client). If a node is labelled dead via our epidemic protocol, we will increment the assigned bucket number
for routing until a live node is found (circular wrap around). If the bucket assigned is the original node, the request
is processed by the application layer itself. A node knows the locations of all the other nodes (live or dead) and can route to them directly.

### Epidemic Protocol & Node Re-joins:

We implemented a hybrid push-pull epidemic protocol. The membership service periodically selects a few random nodes out of the members to pull statuses from.
Pulled timestamps are compared against a threshold to determine if the node is alive or dead. A node that was considered dead earlier due to its timestamp
can re-join the network if it responds to the ping. When a node re-joins the network, the nodes that contain its keys will redistribute the keys it should own
using multiple put requests. Key conflicts are resolved during the redistribution process.
If the pinged node is dead, the request will time out and eventually the membership service will consider the node dead through its timestamp
calculations. Furthermore, each ping request also serves as a "push" as the request being received will update the timestamp of the sender node.

### Replication:

#### PUT
The primary replica submits a PUT request to 3 other replicas to maintain a replication factor of 4.
Replication requests include a timestamp of the request which are used to achieve sequential consistency.

#### REMOVE
On REMOVE requests, the primary replica will submit a REMOVE request to the 3 other replicas
and poll until it receives a SUCCESS response.

#### GET
For GET requests the primary replica directly returns the value requested or null to the client.
For all PUT requests, our re-routing protocol will route the request to the intended "primary" replica.
Therefore, for GET requests we can guarantee that the primary replica will have the most up-to date value.

#### Node Failure and Re-join
During node failures, we maintain a replication factor of 4 keeping track of a live node "window" and appropriately copy
the data to the node joining the "window".
For node re-joins, the successor node of the failed node is responsible for copying data back to the failed node.

