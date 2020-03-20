# A Crawler for the Kademlia-part of the IPFS-network

**For more details, see [our paper](https://arxiv.org/abs/2002.07747). Academic code, run and read at your own risk**

## In a Nutshell

This crawler is designed to enumerate all reachable nodes within the DHT/KAD-part of the IPFS network and return their neighborhood graph.
For each node it saves
* The ID
* All known multiaddresses that were found in the DHT
* Whether it was reachable by the crawler or not, i.e., if a connection attempt was successful
* The agent version.

This is achieved by sending multiple ```FindNode```-requests to each node in the network, targeted in such a way that each request extracts the contents of exactly one DHT bucket.

The crawler is optimized for speed, to generate as accurate snapshots as possible.
It starts from the (configurable) bootstrap nodes, polls their buckets and continues to connect to every peer it has not seen so far.


## Run one or multiple crawls

To run a single crawl simply do:

	make preimages
	make build
	./start_crawl

Note that the preimages only have to be computed *once*.

For multiple crawls, use the `autocrawl.sh` script instead of `start_crawl` in the last line. It takes a duration in days and an optional directory to put logs into.
Note that there will be a lot of output on your disk, one week of crawling (without logs) can lead to 30-50GB of data!
The complete workflow is:

	make build
	./autocrawl [-l logdir] <crawl duration in days>

## Features

### Node Caching

If configured, the crawler will cache the nodes it has seen. The next crawl will then not only start at the boot nodes but also add all previously reachable nodes to the crawl queue. This can increase the crawl speed, and therefore the accuracy of the snapshots, significantly.
Due to node churn, this setting is most reasonable when performing many consecutive crawls.

This property is enabled by default and will store the nodes in a file called ```nodes.cache```.

### Sanity Check ("Canary Peers")

The crawler enumerates the nodes in the network, but without ground truth it is hard to assess the quality and completeness of a crawl.
Therefore, it might be desireable to check whether some known IPFS-nodes appear in the crawl.
If configured, the crawler will check if it has seen the nodes listed in ```configs/cannary.txt```. These can be well-known nodes, such as the ipfs.io-gateway, or self-run nodes, specifically started for the purpose of sanity checking the results.

## Output of a crawl

Two files:
* ```visitedPeers_<start_of_crawl_datetime>_<end_of_crawl_datetime>.csv```
* ```peerGraph_<start_of_crawl_datetime>_<end_of_crawl_datetime>.csv```

The dateformat is dd-mm-yy--H:M:S, where hours are in 24h format. For example, a crawl on the 20th of march 2020 that started at 11:58:17 and ended at 12:04:11:

	visitedPeers_20-03-20--11:58:17_20-03-20--12:04:11.csv

### Format of ```visitedPeers```

Each line in ```visitedPeers``` corresponds to exactly one node on the network. The format is as follows:

	Peer Multihash;[multiaddress_1, multiaddress_2, ..., multiaddress_n];reachable?;agentVersion

Where ```reachable``` is true/false and indicates, whether the respective node could be reached by the crawler or not. Note that the crawler will try to connect to *all* multiaddresses that it found in the DHT for a given peer.
Data example:

	QmQEfD8366rDxNhaUti22p1scVRGNSYcL9YNUZcxktg7sX;[/ip4/49.235.108.57/tcp/4001 /ip6/::1/tcp/4001 /ip4/172.17.0.3/tcp/4001 /ip4/127.0.0.1/tcp/4001];true;go-ipfs/0.4.22/
	
### Format of ```peerGraph```

```peerGraph``` is an edgelist, where each line in the file corresponds to one edge. A line has the form
	
	source multihash;target multihash;<target reachable?>
	
Two nodes are connected, if the crawler found the peer ```target multihash``` in the buckets of peer ```source multihash```.
Example line:

	QmY4N2q3kShvDnMy928SphJRXDEdAcJVDq5sjM5qLsnyHj;QmRXvASjWxqzPFDSvsqyzt9p6DyWNgZ8tVNqgNA4PTw1vk;false
	
which says that the peer with ID ```QmY4N...``` had an entry for peer ```QmRXvAS``` in its buckets and that the latter was not reachable by our crawler.
Therefore, even though the two peers have an active connection (otherwise the latter peer would not be in the buckets of the former peer), the crawler could not connect to the second peer.
Since many nodes reside behind NATs, this is not uncommon to see.

## Bootstrap Peers

By default, the crawler uses the bootstrappeer list provided in ```bootstrappeers.txt```. The file is assumed to contain one multiaddress in each line.
Lines starting with a comment ```//``` will be ignored.
To get the default bootstrap peers of an IPFS node, simply run ```./ipfs bootstrap list > bootstrappeers.txt```.


## Libp2p complains about keylengths

Libp2p uses minimum keylenghts of [2048 bit](https://github.com/libp2p/go-libp2p-core/blob/master/crypto/rsa_common.go), whereas IPFS uses [512 bit](https://github.com/ipfs/infra/issues/378).
Therefore, the crawler can only connect to one IPFS bootstrap node and refuses a connection with the others, due to this key length mismatch.
The environment variable that is used to change the behavior of libp2p is, for some reason, read before the main function of the crawler is executed. So in `start_crawl`, the crawler is started with:

```export LIBP2P_ALLOW_WEAK_RSA_KEYS="" && go run cmd/ipfs-crawler/main.go```
