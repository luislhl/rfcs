- Feature Name: Peer-to-Peer Network
- Start Date: 2019-07-30
- RFC PR: [MR !13](https://gitlab.com/HathorNetwork/rfcs/merge_requests/13)
- Hathor Issue: (leave this empty)
- Author: Pedro Ferreira <pedro@hathor.network>

# Summary
[summary]: #summary

Define how a peer will be identified and the secure protocol of connection between two of them. The Peer-Id is designed in a secure way, so you can trust you are directly connected with whom you are supposed to.

# Motivation
[motivation]: #motivation

The motivation is to have a safe p2p network with: (i) real time communication, (ii) messages broadcasted to all peers, (iii) new peers easily discovered, and (iv) protected from partition attacks.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The communication protocol between two peers is message oriented, and it assumes that all messages arrive ordered. Because of this, we are using TCP as the transport protocol, and we are also using a text-based protocol with EOL to mark the end of each message.

When a brand new full-node is run for the first time, it needs to discover at least one peer of the p2p network to open connection and exchange messages. This process is called bootstrap. After the full-node discovers a few peers, it connects to them, identifies itself, and then starts to exchange messages to both discover other peers and sync the blocks and transactions.

After a connection has been established, there are two steps before getting ready to freely exchange messages (peer A is trying to connect to peer B):

1. The first step is the network configuration. In this step, the connected peers check whether they are running compatible versions of the protocol and the same network configuration (mainnet, testnet-alpha, testnet-bravo, etc). If any error occurs, the connection is closed. This is similar to what Bitcoin does in its [version message][1]

2. The second step is the peer identification. In this step, the peers exchange their Peer-IDs and Entrypoints, then they verify whether they are connected directly to each other (without a man-in-the-middle).

Finally, they are ready to freely exchange messages, i.e., they will exchange their peers, sync their blocks and transactions, and so on.

## Bootstraping

The bootstraping can be done in many different ways. Hathor's full-node uses a DNS query to discover the hosts of the p2p network. After the bootstraping, the full-node connects to all discovered peers and requests information about more peers.

The list of hosts to be DNS queried is hard-coded in the full-node's source code and can be changed passing a different parameter when running the full-node.

Another bootstraping possibility is to pass the URL of the peer you want to connect when starting the node. The connection process is the same of the other case.

## Step 1: Network configuration

This step checks whether the peers are compatible to connect. For instance, a full-node running a testnet configuration cannot connect to a full-node running a mainnet configuration because the DAGs are incompatible, i.e., the genesis is different.

## Step 2: Peer identification

The Peer-ID is important because it allows peers to establish a confidence level for other peers. It may be useful when the network is under attack, and the peers are working together to recover the network.

We use a public key based authentication and we have created a central authority certificate that signs certificates created by the peers and is used to validate the certificates when starting the connection. The Peer-ID is defined as the hash (sha256d) of the public key of the certificate of each peer. To exchange the certificates and establish a secure connection, a Transport Layer Security ([TLS 1.3][2]) will be enabled over the TCP connection.

During this step, the peers exchange their Peer-Id and Entrypoints. Thus, the other part validates the Peer-Id and checks whether it is directly connected to one of the given entrypoints. If the Peer-Id does not match or it is not directly connected, the connection is closed.

Let's say that peer A has opened a connection to peer B. In this case, B must have at least one entrypoint and the connection URL used by A must match one of these entrypoints. But A may have no entrypoints, which means A does not accept connections (e.g. A is behind [NAT][3]). In this case, B does not know whether it is directly connected to A.

If A has no entrypoints, B doesn't close the connection, because we need to accept connections from peers behind NAT but we mark those connections with a flag NO_ENTRYPOINTS, so we know that we couldn't validate if we are directly connected to A because it has no entrypoints.

After these verifications, the connection is ready to freely exchange messages.

## Retry connection

Every time a peer disconnects, its data is kept in a peer storage and a reconnect policy starts. We try to reconnect at most 3 times and multiply the interval time between retries by a multiplier (currently 5). If we still can't reconnect after all the retries, we add a flag `RETRIES_EXCEEDED` to the peer, so we can manually try to reconnect again in the future.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

When a full-node starts, it discovers nodes through bootstraping and then connects to the discovered nodes.

Each new established connection may be in one out of three states: Hello State, Peer-Id State, and Ready State. New connections start in the Hello State.

All messages have the same format: "COMMAND [PAYLOAD]", where the PAYLOAD is optional and different for each command.

## DNS Bootstraping

The full-node does a DNS Query of type TXT to a given list of hosts. Each DNS Query returns a list of URLs where the node should connect to. The URLs have the following format: `scheme://host:port[/?id=<peer-id>]`, where the `id` is optional. When the `id` is given, the full-node should enforce it during the peer identity step and, if not given, we add in this connection the NO_PEER_ID_URL flag to save that we haven't double checked the peer id of this connection.

For each URL resolved in the DNS query, a TLS connection is started and, during the handshake, the peers will exchange their certificates and generate session keys. After the TLS is enabled, both peers will move to the HELLO state.

Another bootstraping possibility is to pass the URL of the peer you want to connect when starting your node using the `--bootstrap` parameter. The connection process is the same of the other case.

## Certificate generation

The first idea was that each peer would create a self signed certificate and exchange them during the TLS connection. However the python wrapper for the openssl lib do not expose the method `SSL_CTX_set_cert_verify_callback` which was an important part to verify the certificates.

Because of that, we've decided to create a central authority certificate and all generated certificates for each peer are not self signed anymore, they are signed by this central authority and have the public key of the peer. This is important to guarantee that the certificate exchange happens when starting the connection.

The central authority certificate can be accessed [here][4] and its private key can be accessed [here][5].

## Man-in-the-middle attack

During the TLS connection, both peers exchange certificates, which are used in an Authenticated Diffie-Hellman Key Exchange, so we can ensure that all future messages exchanged by them are not exposed to a MITM.

If we don't execute the certificate verification, we would be exposed to the following attack, e.g. a MITM (Chuck) can open connection with both sides (Alice and Bob) and manipulate any message.

1. Chuck opens a connection to both Alice and Bob, independently.
2. Chuck identifies as Alice to Bob.
3. Chuck takes the encrypted message from Bob and send to Alice.
4. Chuck takes Alice's reply and forward to Bob.
5. Now, Bob thinks that Chuck is Alice.

Ensuring both certificate and cryptigraphic key exchange is enough to be protected from this kind of attack.

## Hello State

This state is responsible for the Network Configuration. In this state, only two commands are allowed: HELLO and ERROR.

As soon as the peers move to this state, they will exchange a HELLO message.

### HELLO Command

The HELLO command is used to exchange the configuration of the peer. Its payload is a JSON-formatted string in the following format:

```
{
    "app": "Hathor v0.26.0-beta",
    "network": "testnet-bravo",
    "remote_address": "192.168.0.1:51095",
    "genesis_short_hash": "0788746",
    "timestamp": 1564658478,
    "capabilities": [],
    "settings_dict": {
        "P2PKH_VERSION_BYTE": "28",
        "MULTISIG_VERSION_BYTE": "64",
        "MIN_BLOCK_WEIGHT": 21,
        "MIN_TX_WEIGHT": 14,
        "BLOCK_DIFFICULTY_MAX_DW": 0.25,
        "BLOCK_DATA_MAX_SIZE": 100
    }
}
```

The description of each field is:

- `app`: Version of the full-node.
- `network`: The network this full-node is connected to. Anything different from "mainnet" is considered a testnet.
- `remote_address`: Remote IP:PORT seen by this socket.
- `genesis_short_hash`: First 7 chars of the hash of the genesis to prevent syncing incompatible DAGs.
- `timestamp`: Current time according to node's clock. This field can help other nodes to determine that their clock is wrong and is important because nodes will reject blocks that are more than one hour in the future.
- `capabilities`: An array with each capability that the full node supports. For now it's empty but will be used in the future.
- `settings_dict`: A dict containing some of the settings of the full node. The fields that are sent are: `P2PKH_VERSION_BYTE`, `MULTISIG_VERSION_BYTE`, `MIN_BLOCK_WEIGHT`, `MIN_TX_WEIGHT`, `BLOCK_DIFFICULTY_MAX_DW`, and `BLOCK_DATA_MAX_SIZE`. This is an optional field to be sent in the HELLO command but, if presented, should match with the settings of the peer receiving it.

Some validations are executed when the peer receives the HELLO command:

1. All fields must be presented in the payload, except the `settings_dict`;
2. `app`, `network`, `genesis_short_hash`, and `settings_dict` (if presented) values must be the same;

If any validation fails, an ERROR command will be sent and the connection will be closed.

In the case when the `settings_dict` is different, the ERROR command will have a payload, which is the peer's settings.

If all verifications are valid, the peer state moves to PEER-ID.

## Peer-Id State

This state is responsible for the Peer Identification. During this state only two commands are allowed: PEER-ID and ERROR.

### PEER-ID Command

The PEER-ID command is used to exchange the peer identity of the peer. Its payload is a JSON-formatted string in the following format:

```
{
	"id": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
	"pubKey": "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6UelAkULrDmTx3djJhAMZ2p8ziXBGHZW0YZQsmN08mcfCkPJVOtfH0ep8OfhkKkzzBAptsZmOMi0/HZ/ytiyoG6gr4/aot3keh1DL5RqJOeku/L7RyvETOG3zdklz2rf5fgmtdp7EHUqy16imd6pGZzyavyuQx+D1paxn7tVK2fTgvlGIPPWwVC9zsbd9gYCaVw8chDQq3v0xF9FRw2CxDSp2A3zVqbWI+NHfQB7TwMVydFZLl2Z9OO71CfKweNivU7pagaOf49j1jjTXEFRWpUu2w3tUH/QyLz/AJiYdedKhr9a9fGldYu5sAkd07lJPnpq9W32tUe4tJ5eUiOuOQIDAQAB",
	"entrypoints": ["tcp://54.211.192.182:40403"],
}
```

When the PEER-ID command is received, the full-node does some validations before changing the state to READY.

1. Compares the `id` with the sha256d of the public key;
2. If the DNS query return had the `peer_id` parameter on it we validate here if they both match;
3. Validates that the public key of the peer is the same as the one that generated the certificate;
4. Checks whether it is connected to one of the entrypoints (the entrypoint has the format `tcp://name|IP:port`)
  - If the peer is the server, it has access only to the client host and not port entrypoint, so we just validate the host;
  - If the peer is the client, we validate that the URL used to connect is one of the entrypoints;
  - In both cases, if the entrypoint does not have the IP directly (if it's the name), a DNS query must be made to validate if the corresponding URL matches the one connected.

If any vaidation fails, an ERROR command will be sent and the connection will be closed.

If all verifications are valid, the peer sends a READY command to the other peer, to indicates it can move to the next state. If had already received a READY command from the other peer, it moves to the READY state.

### READY Command

The READY command is used to tell the peer you are connected to that you are ready to change to the next state. This command is sent as soon as all PEER-ID validation steps are completed.

This command is needed because in the step 4 of the PEER-ID validation, when a DNS query is needed, it can take a while for one of the peers to validate the connection. So we must change state only after both peers confirm all the above validations.

When changing from the HELLO state to the PEER-ID state this extra care is not needed because the validations are fast, so we guarantee that the new message will be handled only after the Hello message was finished. If we add a slow validation there in the future we would need to handle the state change in a similar way.

After the peer finishes the PEER-ID validation and receives the READY command from the other peer, it changes the state to READY state.

## Ready State

This state is responsible for keeping the connection alive and exchanging information about peers, blocks and transactions on the network. It starts requesting all the peers to the connected node and a ping message loop, to ensure that the connection is still alive. This state allows five commands (besides the sync commands that won't be discussed in this RFC): PING, PONG, GET-PEERS, PEERS, and ERROR.

### PING Command

The PING command is used to verify that the connection is still alive. There is no payload. When the node receives a PING command, it replies with a PONG.

### PONG Command

The PONG command is the response for a received PING command. There is no payload. When the node receives a PONG command, it updates the last_message field of the peer connection with the timestamp.

Using PING and PONG commands we can calculate the RTT between two nodes. We just need to make sure the PONG response is sent as soon as the PING arrives, without any delay and we also need a sequence number in the payload, to know which PING message the PONG is replying.

### GET-PEERS Command

The GET-PEERS command is used to request a list of peers. There is no payload. When the node receives this command, it replies with a PEERS command.

### PEERS Command

The PEERS command is the response when the peer receives a GET-PEERS command and sends a list of known peers. Its payload is a JSON-formatted string in the following format:

```
[
    {
        "id": "214ec65c20889d3be888921b7a65b522c55d18004ce436dffd44b48c117e5590",
        "entrypoints": ["tcp://54.211.192.182:40403"],
        "last_message": 1564658478,
    }
]
```

When the node receives this payload, it update its peer list and if it's not connected to the peer, try to connect, otherwise do nothing. This approach prevents a sybil attack, however we might reach a connection limit, since we are connecting to all the peers without any other option. In the future we will have another RFC that will address this situation and propose a solution where a node can connect only to a subset of all the nodes in the network, also avoiding a possible sybil attack.

# Drawbacks
[drawbacks]: #drawbacks

The current protocol assumes that all messages ordered, i.e., they are delivered in the same order as they are sent. However, we may remove this requirement to allow peers to use simpler transport protocols, such as UDP.

Another drawback is that all connections use TLS 1.3, which creates a secure channel between the peers, but also requires extra processing for each message.


# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- We could use UDP instead of TCP for the transport protocol. This would increase the message exchange throughput, however would increase the complexity of the code, because right now we are assuming that all messages arrive in the correct order (which is not true when using UDP). If we decide to use UDP we could use DTLS (Datagram Transport Layer Security) to use libssl over UDP;

- We could use WebSocket instead of LineReceiver for the messaging protocol but the advantage is not real clear, so we've decided to keep the LineReceiver.

# Prior art
[prior-art]: #prior-art

- Both [Bitcoin][6] and [Ethereum][7] use a message oriented communication protocol with TCP as the transport protocol.
- Ethereum also identifies a peer using its public key, even though they use a different [cryptographic algorithm][8] to generate the node key pair.
- Peer discovery used to be done in [Bitcoin][9] using IRC bootstrapping but is not supported anymore, so when you are connecting to the network for the first time it uses a DNS discovery to find the first node to connect. Ethereum uses a similar approach based in [Kademila][10] in which some nodes are assumed to always being online.
- Both [Bitcoin][1] and [Ethereum][11] exchange a first HELLO (version in bitcoin case) message with basic information of the peer to validate the connection before start exchanging the other messages.
- A list of capabilities of a node is also used by [IMAP][12] and [Ethereum][11].

# Future possibilities
[future-possibilities]: #future-possibilities

- Peer reputation: with a secure method to define a peer id and to estabilish a connection between two peers we could start developing the reputation of one peer, so users can trust more in peers with higher reputation.
- White list and black list of entrypoints. We could define that all base Hathor endpoints are trustworthy and are always in the white list and that any node must be connected to at least one node that is in the white list.
- Define a maximum number of connections per IP address, to prevent a possible attack from the same IP.
- If the peers connections are not estabilished with TLS we should use an Authenticated DH Key Exchange algorithm, so we can sign the messages when exchanging them.
- Persist the list of known and connected peers, so if we need to reconnect in the future we can use this list without the need of a DNS query.
- We should not connect to all available peers. We need an algorithm to select a subset of peers to connect preventing the creation if islands (a set of peers that are isolated from the network, only receiving the data from one connection). Even though we won't connect to all peers, we still need to keep a full list of all peers in the network. Some aspects of a solution were discussed in a [RFC comment][13].
- The PEER-ID command payload should be replaced by a X509 certificate signed. That way the entrypoins, peer_id and pubkey would be put inside the certificate which would be self signed. With this change we would be preventing someone to send tampered peer information in this command. We would just need to check that the peer_id inside the certificate matches the hash of the pubkey used when creating the certificate. We had a brief discussion about it in a [RFC comment][14].


[1]: https://bitcoin.org/en/developer-reference#version
[2]: https://en.wikipedia.org/wiki/Transport_Layer_Security#TLS_1.3
[3]: https://en.wikipedia.org/wiki/Network_address_translation
[4]: https://gitlab.com/HathorNetwork/hathor-python/blob/dev/hathor/p2p/ca.crt
[5]: https://gitlab.com/HathorNetwork/hathor-python/blob/dev/hathor/p2p/ca.key
[6]: https://bitcoin.org/en/developer-reference#p2p-network
[7]: https://github.com/ethereum/devp2p/blob/master/rlpx.md
[8]: https://github.com/ethereum/devp2p/blob/master/discv4.md#node-identities
[9]: https://en.bitcoin.it/wiki/Satoshi_Client_Node_Discovery
[10]: https://en.wikipedia.org/wiki/Kademlia
[11]: https://github.com/ethereum/devp2p/blob/master/rlpx.md#hello-0x00
[12]: https://tools.ietf.org/html/rfc3501#section-7.2.1
[13]: https://gitlab.com/HathorNetwork/rfcs/merge_requests/13#note_204298878
[14]: https://gitlab.com/HathorNetwork/rfcs/merge_requests/13#note_206668807