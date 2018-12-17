- Feature Name: `add_peers_in_runtime`
- Start Date: 2018-11-12
- RFC PR: (leave this empty)
- Sawtooth Issue: (leave this empty)

# Summary
[summary]: #summary This RFC proposes to implement the possibility to add new
peers and remove the existing connections in runtime (through the component
validator endpoint) when a node is working in the `static` peering mode. Along
with that it also adds the corresponding extensions to the off-chain
permissioning model.

# Motivation
[motivation]: #motivation

When an administrator adds a new node to an existing Sawtooth network he/she has
to restart a node with new peering settings. This makes any automation
significantly harder to write than if we had the possibility to add peers in the
runtime and also decreases the uptime.

To resolve this problem our team proposes to add a method to add new peers to a
running node.

An example use case is using Sawtooth along with a service discovery system like
[Consul](https://www.consul.io):

- A Consul node is set up along with the Sawtooth node;
- A middleware continuously fetches the changes of the peer list from the Consul
  node;
- The middleware adds peers to the Sawtooth node via the validator component
  endpoint.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

When an administrator adds new peers to the network you very likely want to add
them without restarting the validator with a newer `--peers` parameter value. To
add a new peer you can send the `ClientAddPeersRequest` to the `component`
endpoint of your Sawtooth validator.

The Protocol Buffers definition for this message is the following:

```protobuf
message ClientAddPeersRequest {
    repeated string peers = 1;
}
```

Simple Python example using `sawtooth_sdk.messaging.stream`:

```python
new_peers = ['tcp://192.168.0.100:8008', 'tcp://192.168.0.200:8008']
add_peers_request = ClientAddPeersRequest(new_peers=new_peers)
future = stream.send(Message.CLIENT_ADD_PEERS_REQUEST,
                     add_peers_request.SerializeToString())
response_serialized = future.result().content
response = ClientAddPeersResponse()
response.ParseFromString(response_serialized)
```

The response Protocol Buffers definition is the following:

```protobuf
message ClientAddPeersResponse {
    enum Status {
        STATUS_UNSET = 0;
        OK = 1;
        INTERNAL_ERROR = 2;
        // One or more of peer URIs were malformed. List of malformed URIs is in
        // the `invalid_uris` field of this response.
        INVALID_PEER_URI = 3;
        MAXIMUM_PEERS_CONNECTIVITY_REACHED = 4;
    }
    Status status = 1;
    repeated string invalid_uris = 2;
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Protocol Buffers definitions
[protobuf]: #protobuf

I propose to add them to `protos/client_peers.proto`:

```protobuf
message ClientAddPeersRequest {
    repeated string peers = 1;
}

message ClientAddPeersResponse {
    enum Status {
        STATUS_UNSET = 0;
        OK = 1;
        INTERNAL_ERROR = 2;
        // One or more of peer URIs were malformed. List of malformed URIs is in
        // the `invalid_uris` field of this response.
        INVALID_PEER_URI = 3;
        MAXIMUM_PEERS_CONNECTIVITY_REACHED = 4;
    }
    Status status = 1;
    repeated string invalid_uris = 2;
}

message ClientRemovePeersRequest {
    repeated string peers = 1;
}

message ClientRemovePeersResponse {
    enum Status {
        STATUS_UNSET = 0;
        OK = 1;
        INTERNAL_ERROR = 2;
        // One of the requested peers do not exist
        PEER_NOT_FOUND = 3;
    }
    Status status = 1;
    repeated string invalid_uris = 2;
}
```

The rationale behind the `invalid_uris` is to be more precise about what is
wrong and to ease the debugging process for developers.

We should also add new message types to `protos/validator.proto`:

```protobuf
message Message {

    enum MessageType {
        // ...
        CLIENT_PEERS_ADD_REQUEST = 131;
        CLIENT_PEERS_ADD_RESPONSE = 132;
        CLIENT_PEERS_REMOVE_REQUEST = 131;
        CLIENT_PEERS_REMOVE_RESPONSE = 132;
        // ...
    }
    // ...
}
```

## How are the requests processed by the validator
[request-processing]: #request-processing

The requests are received on the `component` endpoint. When the validator
receives a new request for adding peers it:

- Validates the format of peer URIs which has to be `tcp://ADDRESS:PORT_NUMBER`
- If the validation was successful then the validator updates its peer list and
  immediately returns the `OK` response. The new peers are connected _after_
  that.

Edge cases:

- The peer address format was wrong in one or more of the provided peers. If
  that happens then the request fails without adding any new peers to the peers
  list and returns the `INVALID_PEER_URI` status along with the list of faulty
  peer URIs.
- If the `--maximum-peer-connectivity` parameter was provided to the validator
  then the validator checks if it has reached the maximum peer connectivity and
  fails with an error if so. The validator also fails if it cannot add _all_ of
  the peers provided in a request without breaking the provided
  `maximum-peer-connectivity`.

# Drawbacks
[drawbacks]: #drawbacks

- The proposed solution does not provide any information on the status of new
  peers. It just returns immediately if a request does not break the conditions
  specified in the previous chapter.
- It does not specify any connection retry policies leaving it to the existing
  peering implementation.

# Rationale and alternatives
[alternatives]: #alternatives

Alternatives were not considered and judging from multiple examples this is the
state-of-the-art solution.

# Prior art
[prior-art]: #prior-art

- [`addnode` in Bitcoin JSON RPC](https://bitcoincore.org/en/doc/0.16.0/rpc/network/addnode/)
- [`admin_addPeer` in Ethereum management API](https://github.com/ethereum/go-ethereum/wiki/Management-APIs#admin_addpeer)

Those two allow adding new peers in their platforms. Interesting points:

- Ethereum can return the connection status;
- Bitcoin allows specifying the connection retry policy.

# Unresolved questions
[unresolved]: #unresolved-questions

- During the pre-RFC discussion on the #sawtooth-core-dev channel, there was no
  final solution on "should it be included to the REST API or no?"
- In the same discussion, there was a proposition to resolve security issues by
  using this feature along with the permissioning module. If we do not add this
  feature to the REST API and hence to the administrative utilities then I do
  not see any point in permissioning because the described feature remains an
  internal interface of the application. Even if we do then we can restrict the
  access to that feature by using a proxy as suggested in the documentation.
- Should the system notify its user about the statuses of new connections or
  leave the status check to the end user?
- Should our team include the integration example of this solution for Consul?