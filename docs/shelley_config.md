``` yaml
# P2P Section explanation
p2p:
  # This is the node's `identifier` in the Poldercast topology. This should be unique and 
  # deterministic (which is not the case in jormungandr) in the poldercast network, and
  # it's highly recommended to be set, because a random `id` is generated on every node restart otherwise.
  # So, we introduced a free sybil attack against to the Poldercast's nodes in default (if it's not set).
  # It simply means that the number of nodes (node ids /w address) in the poldercast increasing by every node's restart
  # and just forgotten very slowly. Therefore, currently there are ~80K node entries in the topology, 
  # which can be one of the reasons the nodes block/slow sometimes.
  #
  # Trick(s): If you choose close numbers for different nodes you run (e.g. xxx01 xxx02 xxx03 etc.), then you will have
  # guaranteed direct connection to two of their closest neighbours.
  # See details in my series:
  # https://gist.github.com/ilap/af8282ff5f81ee5f763fc9994e36b526
  # https://gist.github.com/ilap/e9c43079ba23aa4b6377bea880c8905d#the-hitchhikers-guide-to-the-shelley---chapter-2---the-lock-stock-and-two-smoking-barrel
  public_id: "xxxxxxxxxxxxxxxx"
  
  # Listen address is separeted from `public address` for being able jorm to listen on more than one interfaces.
  # For example on both, the private network (IANA's 10, 172.16, 192.168 etc. address ranges) and the public networ (Internet)
  listen_address: "/ip4/0.0.0.0/tcp/3000"
  
  # This is the public (Internet accessible/faced interface) address, that other nodes can use
  # to connect to your node. This information together /w the `public_id` is gossiped around to the other nodes in
  # the poldercast topology/network to get knowledge about your node. 
  #
  # If it's not set your node will be treated as "unreachable node" like Daedalus wallets behind the NAT,  and 
  # then only the node id is gossiped (no address in the gossup),
  # that means an unreachable node can only be identified by its `node_id`.
  # In this case, your node will be able to connect to other nodes but they would not, and you will see only 
  # OUTBOUND connections only.
  #
  # Tricks: You can set an IPv6 address (whihc is always a public address) and only those nodes would be connected to your node
  # who have the IPv6 configured properly (they can ping/connect to other IPv6 addresses).
  # But, be aware that when a node does not have proper IPv6 configured then it can put you into the quarantinee
  # cos it cannot reach your node, despite you have public_Address set.
  public_address: "/ip4/xxx.xxx.xxx.xxx/tcp/yyyy"
  
  # This is the maximum INBOUND and OUTBOUND connections that jormungandr handles.
  # INBOUND connections only applies when our node is gossipped accross to the whole network, means other nodes
  # have a knowledge about your node and therefore they can connect you (See, above public address).
  #
  # OUTBOUND connections: Initially, our nodes relies on the `trusted peers`, if they're not set and we are unknown 
  # in the poldercast network (meaning never connected or forgotten by it), then we're out of luck as
  # 1. we cannot connect to anybody (as not rusted peers are set) and
  # 2. nobody would connect to us as they do not know our public address or it's forgotten.
  #
  # Trick(s): I would set as high as you can, To be more precise, at least to the number of all available and on-line nodes exist in the poldercast network - 1. 
  # Why n-1 number? cos, in theory, if you're connected to a node that's initiated by you, then that node should not initiate
  # a connection to you as you have a proper bidirectional gRPC channel established.
  # if it less then that nr. then your node will be quarantined by other nodes who cannot connect to you and your 
  # node does not have establishment connection, of course it only applies if your node have public address set, 
  # but are not reachable due some reasons, when the other node want initiate a connections.
  max_connections: 1000
  
  
  # In poldercast newtork, there are nodes that do not have public_address set for mainly two reasons:
  # 1. They're behind the NAT, for example Daedalus wallets in a soho environment.
  # 2. They did not set the the public_address, to eliminate incomming connections (bad behaviour).
  # So, wallets are connecting to nodes that have `public_ip` set, 
  #
  # Tricks: If our node is stable, then I would set it some high for example 32 or 64, to help the wallets be able to connect to your node.
  max_unreachable_nodes_to_connect_per_event: 20
  
  # If it's set then it resets __only__ the:
  # - `available` nodes: nodes in `all` nodes that are not quarantined and reachable.
  # - `quarantined` nodes: nodes in `all` nodes that are quarantined.
  # - `unreachable` nodes: nodes in `all` nodes that do not have public_id set.
  # of the `all` nodes (all ~80K nodes, i.e. union of that 3 type of nodes in  the topology), __BUT__ not the all nodes,
  # so that's just keep growinug and hardly forget individual nodes in it.
  #
  # Tricks: No any tricks atm, as the all nodes are not reset, meaning not too much help from it until
  # it's fixed in the jorm's code. Meaning, you will always rebuild the available, quarantined and unreachable from
  # that huge number of `all` nodes.
  topology_force_reset_interval: 30m
  
  # This is the intervall where the information (node_id, public_address) of 10 selected nodes (gossip) of the poldercast topology/network 
  # are sent to the 44 + `max_unreachable_nodes_to_connect_per_event` number of nodes in every `gossip_interval` seconds.
  #
  # Trick(s): default is 10s, but I could see that thise gossips are just sent gradually (instead of one batch in every 10sec)
  # due to the fact that the node is exhausted, meaning cannot process it as a batch.
  #
  # This should be low value, due to the fact of high `churn` (nodes leaving and joining) of the network.
  gossip_interval: 10s
  
  policy:
    # This tells to jorm that how many minutes a node will be quarantined before it lifted up to the `available` or `unreachable` nodes.
    # Trick(s) set it low, cos in that case the restarted nodes that have public_id set can be lifted at quickly.
    quarantine_duration: 5m
  topics_of_interest:
    # Poldercast is a Topic Based Pub/Sub (TBPS) System, and jormungandr use two : 
    # - `Blocks` topic, for block announcements and 
    # - `Messages` topic, for others (fragments/txs, solicitor etc)
    #
    # Tricks(s): High means, the poldercast would like to send topics to those are in common, means to the nodes in which
    # the priority is similarly high.
    blocks: high
    messages: high
  
  # The trusted peers when your node initially contact to for getting:
  # 1. The genesis if you do not have it in the DB
  # 2. The tip
  # 3. and the blocks you have till the tip.
  #
  # Trick(s): You can have some of your other nodes or other reliable pools' public_id and public_address set.
  # Or even you can leave it empty, if your node is already introduced to and knowledged by the poldercast network,
  # as the other nodes will be trying to contact to your node anyway.
  # Also, you can set an IPv6 address too to elinimate any noisy/messy nodes, as not too many nodes use IPv6
  # and who can set it up (if it's not set already by ISPs), then their have some knowledge in the IT/CS field.
  trusted_peers:
          #    - address: "/ip4/xxxxx/tcp/yyyy"
          #  id: xxxxxxxx

# Ths sqlite3 database for storing Blocks and related infotmations:
# - Blocks: Contains the hash and the block itself
# - BlockInfo: It contains, the information of the blocks (block date as depth, parent etc)
# - Tags: In which the HEAD column is the tip/head of the blockhain (in point-of-view of the current node, whihc can be false)
#
# Trick(s): Sqlite3 is an embedded sql that is capable for concurent writes/read from different processes in the same VM/VPS/Phys server.
# Therefore, it can be shared between nodes if jorm version is <= 0.8.5, cos I have not checked the code for the above releases.
# That means you can set up a 3 nodes, as an example, that use the same storage, from which one node currently does not have 
# trusted peers set (it must be known by the network already), and just reading/writing the tips from that DB.
# While the other two, have trusted peers sets and their main job is feeding the DB /w blocks and their tip.
storage: "/data/home/shelley/clust/storage"
```
