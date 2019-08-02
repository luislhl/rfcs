- Feature Name: Peer-to-Peer Network
- Start Date: 2019-07-30
- RFC PR: [MR !12](https://gitlab.com/HathorNetwork/rfcs/merge_requests/12)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

Define how a peer will be identified and the secure protocol of connection between two of them. The Peer-Id is designed in a secure way, so you can trust the peer you are connecting to.

# Motivation
[motivation]: #motivation

The motivation is to increase security in the peer connections and real time communication. 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The communication protocol between two peers is message oriented, and it assumes that all messages arrive ordered. Because of this, we are using TCP as the transport protocol, and we are also using a text-based protocol with EOL to mark the end of each message.

When a brand new full-node is run for the first time, it needs to discover at least one peer of the p2p network to open connection and exchange messages. This process is called bootstrap. After the full-node discovers a few peers, it connects to them, identify itself, and then start to exchange messages to both discover other peers and sync the blocks and transactions.

After a connection has been established, there are two steps before getting ready to freely exchange messages (peer A is trying to connect to peer B):

1. The first step is the network configuration. In this step, the connected peers checks whether they are running compatible versions of the protocol and the same network configuration (mainnet, testnet-alpha, testnet-bravo, etc). If any error occurs, the connection is closed.

2. The second step is the peer identification. In this step, the peers exchange their Peer-IDs and Entrypoints, then they verify whether they are connected directly to each other (without a man-in-the-middle).

Finally, they are ready to freely exchange messages, i.e., they will exchange their peers, sync their blocks and transactions, and so on.

## Bootstraping

The bootstraping can be done in many different ways. Hathor's full-node uses a DNS query to discover the hosts of the p2p network. After the bootstraping, the full-node connects to all discovered peers and requests information about more peers.

The list of hosts to be DNS queried is hard-coded in the full-node's source code and can be changed passing a different parameter when running the full-node.

## Step 1: Network configuration

This step checks whether the peers are compatible to connect. For instance, a full-node running a testnet configuration cannot connect to a full-node running a mainnet configuration because the DAGs are incompatible, i.e., the genesis is different.

## Step 2: Peer identification

The Peer-ID is important because it allows peers to establish a confidence level for other peers. It may be useful when the network is under attack, and the peers are working together to recover the network.

As there is no central authority to distribute certificates to the peers, we chose to use a self-generated certificate for the peers, and the Peer-ID is defined as the hash (sha256d) of the public key of the certificate. To exchange the certificates and establish a secure connection, a Transport Layer Security (TLS) will be enabled over the TCP connection.

During this step, the peers exchange their Peer-Id and Entrypoints. Thus, the other part validates the Peer-Id and checks whether it is directly connected to one of the given entrypoints. If the Peer-Id does not match or it is not directly connected, the connection is closed.

Let's say that peer A has opened a connection to peer B. In this case, B must have at least one entrypoint and the connection URL used by A must match one of these entrypoints. But A may have no entrypoints, which means A does not accept connections (e.g. A is behind a NAT). In this case, B does not know whether it is directly connected to A.

If A has no entrypoints, B doesn't close the connection, but it marks the connection with the flag ENTRYPOINT-VERIFICATION-FAILS.

After these verifications, the connection is ready to freely exchange messages.


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a full-node starts, it discovers nodes through bootstraping and then connects to the discovered nodes.

Each new established connection may be in one out of three states: Hello State, Peer-Id State, and Ready State. New connections start in the Hello State.

All messages have the same format: "COMMAND [PAYLOAD]", where the PAYLOAD is optional and different for each command.

## DNS Bootstraping

The full-node does a DNS Query of type TXT to a given list of hosts. Each DNS Query returns a list of URLs where the node should connect to. The URLs have the following format: `scheme://host:port[/?id=<peer-id>]`, where the `id` is optional. When the `id` is given, the full-node should enforce it during the peer identity step.

## Hello State

This state is responsible for the Network Configuration. In this state, only two commands are allowed: HELLO and ERROR.

### HELLO Command

The HELLO command is used to exchange the configuration of the peer. Its payload is a JSON-formatted string in the following format:

```
{
    "app": "Hathor v0.26.0-beta",
    "network": "testnet-bravo",
    "remote_address": "192.168.0.1:51095",
    "genesis_short_hash": "0788746",
    "timestamp": 1564658478,
    "settings_hash": "f9d86028c6e0d64e225186f96acb69338b2c59764df79162107f5c4bb34d1310",
}
```

The description of each field is:

- `app`: Version of the full-node.
- `network`: The network this full-node is connected to. Anything different from "mainnet" is considered a testnet.
- `remote_address`: Remote IP:PORT seen by this socket.
- `genesis_short_hash`: First 7 chars of the hash of the genesis to prevent syncing incompatible DAGs.
- `timestamp`: Current time according to node's clock. This field can help other nodes to determine that their clock is wrong and is important because nodes will reject blocks that are more than one hour in the future.
- `settings_hash`: Hash of a dict containing some of the settings of the full node. The fields that are considered are: `P2PKH_VERSION_BYTE`, `MULTISIG_VERSION_BYTE`, `MIN_BLOCK_WEIGHT`, `MIN_TX_WEIGHT`, `BLOCK_DIFFICULTY_MAX_DW`, and `BLOCK_DATA_MAX_SIZE`. The hash is calculated in the following method (d is the dict with the settings):

```
    settings_hash = hashlib.sha256(json.dumps(d).encode('utf-8')).digest()
```

If `settings_hash` is different in both peers, an ERROR command will be sent to the peer who sent the HELLO with the settings and the connection will be closed.

## Peer-Id State

This state is responsible for the Peer Identification. So, it starts making a `STARTTLS` to enable TLS. During the TLS handshake, the peers will exchange their certificates and generate session keys. After the TLS has been enabled, only two commands are allowed: PEER-ID and ERROR.

### PEER-ID Command

The PEER-ID command is used to exchange the peer identity of the peer. Its payload is a JSON-formatted string in the following format:

```
{
	"id": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
	"entrypoints": ["tcp:54.211.192.182:40403"],
}
```

When the PEER-ID command is received, the full-node compares the `id` with the sha256d of the public key. Then, it checks whether it is connected to one of the entrypoints. If the `id` and entrypoints checks are valid, the peer state changes to READY.

## Ready State

This state is responsible for keeping the connection alive and exchange information about peers, blocks and transactions on the network. It starts requesting all the peers to the connected node and a ping message loop, to assure that the connection is still alive. This state allows five commands (besides the sync commands that won't be discussed in this RFC): PING, PONG, GET-PEERS, PEERS, and ERROR.

### PING Command

The PING command is used to verify that the connection is still alive. There is no payload. When the node receives a PING command, it replies with a PONG.

### PONG Command

The PONG command is the response for a received PING command. There is no payload. When the node receives a PONG command, it updates the last_message field of the peer connection with the timestamp.

Using PING and PONG commands we can calculate the RTT between two nodes. We just need to make sure the PONG response is sent as soon as the PING arrives, without any delay.

### GET-PEERS Command

The GET-PEERS command is used to request a list of peers. There is no payload. When the node receives this command, it replies with a PEERS command.

### PEERS Command

The PEERS command is the response when the peer receives a GET-PEERS command and sends a list of known peers. Its payload is a JSON-formatted string in the following format:

```
[
    {
        "id": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
        "entrypoints": ["tcp:54.211.192.182:40403"],
        "last_message": 1564658478,
    }
]
```

When the node receives this payload, it update its peer list and if it's not connected to the peer, try to connect, otherwise do nothing. This approach prevents a sybil attack, however we might reach a connection limit, since we are connecting to all the peers without any other option. In the future we will have another RFC that will approach this situation and propose a solution where a node can connect only to a subset of all the nodes in the network, also avoiding a possible sybil attack.

# Drawbacks
[drawbacks]: #drawbacks

The current protocol assumes that all messages ordered, i.e., they are delivered in the same order as they are sent. However, we may remove this requirement to allow peers to use simpler transport protocols, such as UDP.

Another drawback is that all connections to use TLS 1.3, which creates a secure channel between the peers, but also requires extra processing for each message.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could use UDP instead of TCP for the transport protocol. This would increase the message exchange speed, however would increase the complexity of the code, because right now we are assuming that all messages arrive in the correct order (which is not true when using UDP). If we decide to use UDP we could use DTLS (Datagram Transport Layer Security) to use libssl over UDP;

- We could use WebSocket instead of LineReceiver for the messaging protocol but the advantage is not real clear, so we've decided to keep the LineReceiver.


# Prior art
[prior-art]: #prior-art

-

# Unresolved questions
[unresolved-questions]: #unresolved-questions

-

# Future possibilities
[future-possibilities]: #future-possibilities

- Peer reputation: with a secure method to define a peer id and to estabilish a connection between two peers we could start developing the reputation of one peer, so users can trust more in peers with higher reputation.
- White list and black list of entrypoints. We could define that all base Hathor endpoints are trustworthy and are always in the white list and that any node must be connected to at least one node that is in the white list.
- Define a maximum number of connections per IP address, to prevent a possible attack from the same IP.

# References
[references]: #references