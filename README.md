# IPFS-Link

1. Publish dynamic multiaddresses of an isolated or private IPFS node using publicly resolvable IPNS.
2. Get multiaddresses to such a node that has published its multiaddresses using IPNS.

To understand the purposes of this tool better, see [Use cases](#use-cases).

#### Keywords

ipfs ; ipns ; private-network ; bandwidth savings ; multiaddress ; swarm key ; dynamic IP address ; port-forwarding ; proxy ; reverse proxy ; pubsub ; DDNS ; p2p ; home server ; exposing localhost ; bootstrap node ;

## Use cases

#### Intro

Before considering the following use cases let us take note of the fact that IPFS nodes are multipurpose. They are not just filesystem nodes, but can also do [TCP port forwarding](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p), [HTTP proxying](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#p2p-http-proxy) and [pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-pubsub). Therefore, using IPFS nodes you can not only do file-sharing, but also all sorts of p2p streams like chatting, gaming, screen sharing, RDP/VNC, SSH/remote shell and what not. You can expose your local server through an IPFS node. You can also connect to IPFS nodes from browser using WebSocket or use the node to pass on signaling data for WebRTC to connect remote browsers. IPFS nodes may also be used for peer discovery. Using the [autorelay](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#autorelay) feature, IPFS nodes can give you NAT-traversal, when your machine is inaccessible from the public internet.

Therefore, you may have enough reasons to run a closed private network of IPFS nodes, or just run a personal but open, i.e. not private, IPFS node that only you and your friends can connect to.

#### Private network - Dynamic IP address

Say, you have a bootstrap node in a private IPFS network. You need to tell other peers in that network its multiaddresses so that they can connect to it. But your node doesn't have a static public IP address. The traditional way to tackle this would be to use a [DDNS](https://en.wikipedia.org/wiki/Dynamic_DNS) service. But you don't need to go through that trouble anymore. `ipfs-link` publishes the corresponding dynamic multiaddresses at the IPNS key `/ipns/<your node ID>`. Other peers can then get those multiaddresses simply by using `ipfs-link <your node ID>`.

#### Private network - Swarm key derived from node ID

You have an IPFS node running at your home, on a Raspberry Pi may be. You want it to be private, i.e. have a swarm key, such that only you can connect to it from elsewhere. But you don't want to carry around the swarm key and risk losing it. You want to carry just the node ID. `ipfs-link -k hash <node ID>` gives you the swarm key and `ipfs-link <node ID>` gives you the multiaddress to connect to.

#### Isolated node - Bandwidth savings with [Routing.Type=none](https://github.com/ipfs/go-ipfs/blob/master/docs/config.md#routingtype)

IPFS nodes, even as DHT clients, [consume a lot of bandwidth](https://github.com/ipfs/go-ipfs/issues/3429). This is mostly due to the connection with the [public WAN DHT](https://docs.ipfs.io/concepts/dht/#dual-dht). But perhaps you just need to host a node for port-forwarding or (reverse) proxying purposes and you don't really need the public WAN DHT. However, you don't have a public IP address so you would need NAT-traversal. Hence, you need to use the autorelay feature. Because you need to connect to a public relay, you can neither get rid of the bootstrap nodes, nor can you make the node private using a swarm key. To solve this, `ipfs-link` gets your node online with no connection to the WAN DHT, thus saving hugely on bandwidth. Once your node has found a relay, it publishes the corresponding public multiaddress(es) over IPNS. Whoever needs to connect to your node can get those addresses simply using `ipfs-link <your node ID>`.

## Working principle

The isolated or private node cannot connect to the [public WAN DHT](https://docs.ipfs.io/concepts/dht/#dual-dht). So, even if it publishes its multiaddresses over IPNS, public gateways such as https://ipfs.io cannot access those records. But, what if, the node saves its IPNS records in the local filesystem that another *short-lived* IPFS node on the same machine can then access and (re)publish? This second node may be called an *aux*(illiary) node. The aux node doesn't need to connect to the main node, as it accesses the latter's IPNS records from the disk. Aux node is well-connected to the public WAN DHT and has [IPNS over pubsub](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipns-pubsub) enabled for faster resolution of IPNS records published by it. Aux node stays online only for a few seconds, just enough to publish the main node's iPNS records. It then shuts down. After 15 mins, it's up again to do the same job and so on... All this is driven and managed by the `ipfs-link` script. Because the aux node is so short-lived or *ephemeral*, its bandwidth usage is minimal. The main node is detached from the public WAN DHT, so it's bandwidth usage is minimal too.

You may now ask, well, isolated as it is from the WAN DHT, can our main node publish over IPNS at all? Turns out it can - if it has IPNS over pubsub enabled.

## Usage

```shell
ipfs-link [option] [<peerID>]
```

- Provide peerID only when seeking multiaddress to (or swarm key of) the corresponding node (see [Examples](#examples) below).
- When no peerID is provided, `ipfs-link` launches the IPFS node with repository at <u>path passed by `-c` option</u> or <u>the environment variable: `IPFS_PATH`</u> or <u>the default: `${HOME}/.ipfs`</u>. So, no need to launch the node manually with `ipfs daemon`.

#### Option

`-k hash | rand`

​	For swarm key generation. `hash` implies key derived from peerID. `rand` implies random key.

`-c <path>`

​	Pass path to local IPFS repository. Can also use `IPFS_PATH` environment variable instead. If absent, the default path `~/.ipfs` is assumed.

`-v`

​	Version

`-h`

​	Show usage

#### Examples

- Get node with repo(sitory) at `${path}` online and publish its multiaddress via IPNS: 

  ```shell
  ipfs-link -c "${path}"
  ```

  This launches the node using `ipfs daemon` with pubsub, IPNS over pubsub and auto-GC enabled. If there is no `swarm.key` in the repo, the node is launched with `--routing=none`, isolating it from WAN DHT and saving on bandwidth. If autorelay has been enabled (prior to executing `ipns-link`) with `ipfs config --bool Swarm.EnableAutoRelay true`, the isolated node will still get a relay, albeit after a little delay.

- Get node with repo at `${path}` online with a random swarm key and publish its multiaddress via IPNS: 

  ```shell
  IPFS_PATH="${path}" ipfs-link -k rand
  ```

  The node is made online as part of a private network. The swarm key is saved inside the repo.

- Get node with repo at `${path}` online with a deterministic swarm key derived from the node ID and publish its multiaddress via IPNS: 

  ```shell
  export IPFS_PATH="${path}"
  ipfs-link -k hash
  ```

  Use this when the node ID itself is a secret, shared between trusted parties only. Use this for example with your home server that only you and your friends shall connect to.

- Get multiaddress to a remote node with ID=`${id}`, that has been launched with `ipns-link`: 

  ```shell
  ipfs-link "${id}"
  ```

- Get swarm key to a node (home server for example) that was launched with a swarm key derived from the node ID=`${id}`: 

  ```shell
  ipfs-link -k hash "${id}"
  ```

## Bug-reports and Feedbacks

Post at [issues](https://github.com/SomajitDey/ipfs-link/issues) and [discussion](https://github.com/SomajitDey/ipfs-link/discussions), or [write to me](mailto://hereistitan@gmail.com).

------

###### [GNU GPL v3-or-later](https://github.com/SomajitDey/ipfs-link/blob/main/LICENSE) &copy; 2021 Somajit Dey

