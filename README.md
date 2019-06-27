actix-raft
==========
An implementation of the Raft consensus protocol using the actix framework.

This differs from other Raft implementations in that it is fully reactive and embraces the async ecosystem.

- It is driven by actual Raft related events taking place in the system as opposed to being driven by a `tick` operation.
- Integrates seamlessly with other actors by exposing a clean set of message types which must be used to interact with the Raft actor from this crate, thus providing a more simple and clear API.
- This Raft implementation provides the core Raft consensus protocol logic, but provides clean integration points for efficiently implementing the networking and storage layer.
- Uses actix traits (actors & handlers) as integration points for users to implement the networking and storage layers. This allows for the storage interface to be synchronous or asynchronous based on the storage engine used, and allows for easily integrating with the actix ecosystem's networking components for efficient async networking.
- Fully supports dynamic cluster membership according to the Raft spec. See the [`ProposeConfigChange`](#proposeconfigchange) admin command defined below.
- Does not impose buffering of outbound messages, but still allows for it if needed.
- Uses [Prost](https://github.com/danburkert/prost) for simple and clean protocol buffers data modelling.

This implementation strictly adheres to the [Raft spec](https://raft.github.io/raft.pdf) (*pdf warning*), and all data models use the same nomenclature found in the spec for better understandability.

### admin controls
Actix-raft nodes may be controlled in various ways outside of the normal flow of the Raft protocol using the `AdminCommand` message type. This allows the parent application — within which the Raft node is running — to influence the Raft node's behavior based on application level needs. The most common admin command which applications will use is quite like the `ProposeConfigChange`. See the API documentation for more details. Here are a few of the admin commands available.

#### `InitWithConfig`
When a Raft cluster is booted for the very first time, all nodes will refuse to become leader as they have no peers to communicate with. The parent application must be aware of this condition, and by way of its service discovery meachanism (or any other means), the parent application must submit an `InitWithConfig` command to the Raft node with the initial cluster config. This command will fail if the node is not a follower, if the node's log index is not `0`, or if the node has a set of configured peers already.

This will cause the Raft node to immediately perform the election protocol. It will increment the current term, and will then submit `RequestVote` RPCs to all of the nodes in the given config list. It is safe for every node in the cluster to perform this action when it is initially started, as the Raft protocol's randomization factor for election timeouts ensures that a leader will eventually be elected.

Once this process has been completed, the newly elected leader will append the given config change data to the log, ensuring that the new configuration will be reckoned as the initial cluster configuration moving forward.

**NOTE WELL:** when developing an application based on this Raft implementation, it is important to pay attention to the initial cluster formation phase. All Raft nodes will startup in a passive state refusing to become the leader until an election is forced with the `InitWithConfig` command. In order to ensure that multiple independent clusters aren't formed by prematurely issuing the `InitWithConfig` command before all peers are discovered, it would be prudent to have all discovered node's exchange some information during their handshake protocol. This will allow the parent application to make informed decisions as to whether the `InitWithConfig` should be called and how early it should be called when starting a new cluster. An application level configuration for this facet is recommended.

As a rule of thumb, when new nodes come online, the leader of an existing Raft cluster will eventually discover the node (via the application's discovery system), and in such cases, the application should submit a new `ProposeConfigChange` to the leader to add it to the cluster.

#### `InitAsLeader`
When a Raft node is booted for the very first time, this command will attempt to force the node to assume the Raft leader role. If the Raft node is running as part of a live cluster with other members, or is not currently a follower, then this command will **always fail**, as following through with such a command would critically violate Raft's safety gurantees. Instead, this command is useful for running a standalone deployment, which some applications may want to support. If your application does not need such a behavior, then this command is not going to be very useful for you.

After a node has been running and making progress in standalone mode, if a config change is proposed, there are a few critical invariants which the parent application must uphold:

- the newly added nodes must not have been running in standalone mode, as this will lead to data loss.
- the original node must remain online until at least half of the other new nodes have been brough up-to-date, otherwise the Raft cluster will not be able to make progress. After the other nodes have been brought up-to-date, everything should run normally according to the Raft spec.

#### `ProposeConfigChange`
This command will propose a new config change to a running cluster. This command will fail if the Raft node which this command was submitted to is not the Raft leader. Once the leader receives this command, the new configuration will be appended to the log and the Raft dynamic configuration change protocol will begin. For more details on how this is implemented, see §6 of the Raft spec.

Cluster auto-healing, where cluster members which have been offline for some period of time are automatically removed, is an application specific behavior, but is fully supported via this dynamic cluster membership system.

### snapshots
The snapshot capabilities defined in the Raft spec are fully supported by this implementation. The storage layer is left to the application which uses this Raft implementation, but all snapshot behavior defined in the Raft spec is supported, including:

- Configurable snapshot policies. This allows nodes to perform log compacation at configurable intervals.
- Leader based `InstallSnapshot` RPC support. This allows the Raft leader to make determinations on when a new member (or a slow member) should receive a snapshot in order to come up-to-date faster.

The API documentation is comprehensive and includes implementation tips and references to the Raft spec for further guidance.
