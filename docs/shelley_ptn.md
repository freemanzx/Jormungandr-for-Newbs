# Install & Config Shelley's Haskell Private Testnet

## Install prerequisites

``` bash
$ curl -sSL https://get.haskellstack.org/ | sh
...

$ sudo apt install pkg-config libsystemd-dev libtinfo-dev
...
The following NEW packages will be installed:
  slack

$ git clone https://github.com/input-output-hk/cardano-node.git
Cloning into 'cardano-node'...
...
```

## Build & Install `cardano-node` and `cardano-cli`

``` bash
$ cd cardano-node/cardano-node && stack build && stack install && . ~/.profile`
```
### Update

``` bash
cd ~/cardano-node && git pull -r && stack build && stack install
Copying from ...

Copied executables to /home/ilap/.local/bin:
- cardano-cli
- cardano-node
- cardano-tx-generator
- chairman
```

## Config PTN

``` bash
$ mkdir -p ~/ptn/{config,data,db}
```

### Download `node.config` and `genesis.json` template files

``` bash
$ cd ~/ptn/config && 
curl -o pbft_config.json https://raw.githubusercontent.com/cardano-community/guild-operators/master/files/cnode_config.yaml.sample &&
curl  https://raw.githubusercontent.com/cardano-community/guild-operators/master/files/genesis.json | jq '.' > ~/ptn/pbft_genesis.json
...
# It generates random NodeID:
# We do not need NodeID, and I think NumberOfCoreNodes too.
# -e "s#NodeId:.*#NodeId:`od -A n -t u8 -N 8 /dev/urandom`#" \

$ sed -i -e "s#GenesisFile:.*#GenesisFile: `pwd`/pbft_genesis.json#" ~/ptn/config/pbft_config.json
```

### Create `node.config` file (Optional)

__DO NOT DO THIS__ if you use the downloaded one from the previous step.

``` bash
$ cd ~/ptn/config && cat > node.config <<EOF
# Node config
# It also generates the initial random node ID.
# Not Required for PBFT NodeId: `od -A n -t u8 -N 8 /dev/urandom`
# Not required for PBFT NumCoreNodes: 1
PBftSignatureThreshold: 1000000000000000
GenesisFile: `pwd`/pbft_genesis.json
Protocol: RealPBFT
RequiresNetworkMagic: RequiresNoMagic
TurnOnLogMetrics: False
TurnOnLogging: True
ViewMode: SimpleView

# Update Params
ApplicationName: cardano-sl
ApplicationVersion: 1
LastKnownBlockVersion-Major: 0
LastKnownBlockVersion-Minor: 2
LastKnownBlockVersion-Alt: 0

# Logging

# global filter; messages must have at least this severity to pass:
minSeverity: Info

# if not indicated otherwise, then messages are passed to these backends:
defaultBackends:
  - KatipBK

# if not indicated otherwise, then log output is directed to this:
defaultScribes:
  - - StdoutSK
    - stdout

# more options which can be passed as key-value pairs:
options:
  # Disable "Critical" logs that are actually metrics...
  mapBackends:
    cardano.node-metrics: []
    cardano.node.BlockFetchDecision.peers: []
    cardano.node.ChainDB.metrics: []
    cardano.node.metrics.ChainDB: []
    cardano.node.metrics: []
    cardano.node.metrics: []

# these backends are initialized:
setupBackends:
  - KatipBK

# here we set up outputs of logging in 'katip':
setupScribes:
  - scName: stdout
    scKind: StdoutSK
    scFormat: ScText
EOF
```

### Create singing and verifying keys

For BFT members, __DO NOT GENERATE KEY__, use the given one by `Priyank`.

``` bash
$ cardano-cli keygen --real-pbft --secret ~/ptn/data/pbft0.key --no-password
```

### Create certs

Certs is generated/derived from the generated/given `pbft0.key` above.

``` bash
$ cardano-cli to-verification --real-pbft --secret  ~/ptn/data/pbft0.key --to pub.key


# Create the verification key
$ cardano-cli to-verification --real-pbft --secret ~/ptn/data/pbft0.key --to ~/ptn/data/pbft0.vfk

# Check the verification/public key
$ cardano-cli signing-key-public --real-pbft --secret ~/ptn/data/pbft0.key | awk '/base64/ { print $4}'
HyZ+SQ3odbYPHp8OyIk/nbkzbWbOXbVYmcS4mIWKoUZGiZi6b1Fb9334nvL0sgOktB7dlKnSvUcsWOQpAFFzTA==
$ cat ~/ptn/data/pbft0.vfk
HyZ+SQ3odbYPHp8OyIk/nbkzbWbOXbVYmcS4mIWKoUZGiZi6b1Fb9334nvL0sgOktB7dlKnSvUcsWOQpAFFzTA==
```

Only For BFT members, __DO NOT GENERATE KEY__!
```
# Create the cert
# It's not necessary as the certs are in the genesis file.
# DO NOT CREATE CERT, DERIVE FROM GENESIS.
$ cardano-cli issue-delegation-certificate \
--config ~/ptn/config/pbft_config.json \
--since-epoch <?> \
--secret <THE ISSUER SECRET/KEY FOR GENESIS> \
--delegate-key ~/ptn/data/pbft0.key \
--certificate ~/ptn/data/pbft0.cert

# THIS ONLY WORKS, IF YOUR verification key is in the genesis file. To check:
grep `cat ~/ptn/data/pbft0.vfk` ~/ptn/config/pbft_genesis.json
"delegatePk": "HyZ+SQ3odbYPHp8OyIk/nbkzbWbOXbVYmcS4mIWKoUZGiZi6b1Fb9334nvL0sgOktB7dlKnSvUcsWOQpAFFzTA==",


# Extract cert from genesis.
grep `cat ~/ptn/data/pbft0.vfk` -B 3 -A 2 ~/ptn/config/pbft_genesis.json | sed -e 's@^.*{@{@' -e 's@^.*},@}@' > ~/ptn/data/pbft0.cert
```

## FnF Key generations
We need some addressese for:
1. Pool and for
2. depositing fund i.e. address

### Depositing fund

``` bash
## General sheley key
cardano-cli shelley address key-gen --verification-key-file $KEY_NAME.vkey --signing-key-file $KEYNAME.skey
cardano-cli shelley address build  --payment-verification-key-file $KEY_NAME.vkey | tee address "$KEY_NAME.addr"
```

## Run (Private) PTN

Get the latest topology file from [Guild's Wiki](https://github.com/cardano-community/guild-operators/wiki/Topology-for-Network-used-by-cardano-node-on-PTN) 

``` bash
$ cat > ~/ptn/config/pbft_topology.json <<EOF
{
  "Producers":[
     {
        "addr":"51.79.141.170",
        "port":9000,
        "valency":1
     },     {
        "addr":"88.99.83.86",
        "port":9000,
        "valency":1
     },
     {
        "addr":"139.99.237.20",
        "port":9000,
        "valency":1
     },
     {
        "addr":"139.99.237.20",
        "port":9001,
        "valency":1
     }
  ]
}
EOF
```

### For Active node

``` bash
# Run the node
# v1.9.1 does not use it anymore --genesis-hash `cardano-cli print-genesis-hash --genesis-json pbft_genesis.json` \
# v1.9.1 does not use it anymore--genesis-file ~/ptn/config/pbft_genesis.json \
$ cardano-node run \
          --config ~/ptn/config/pbft_config.json \
          --database-path ~/ptn/db \
          --host-addr `curl ifconfig.me` \
          --signing-key ~/ptn/data/pbft0.key \
          --delegation-certificate ~/ptn/data/pbft0.cert \
          --port 9000 \
          --socket-path ~/ptn/data/pbft_node.socket \
          --topology ~/ptn/config/pbft_topology.json
```


### For Passive node

``` bash
$ cardano-node run \
  --config ~/ptn/config/pbft_config.json \
  --database-path ~/ptn/db \
  --host-addr `curl ifconfig.me` \
  --port 9000 \
  --topology ~/ptn/config/pbft_topology.json
```

## Create addresses from key.

``` bash
$ cd ~/ptn/config &&
cardano-cli signing-key-address --real-pbft --testnet-magic `jq '.protocolConsts.protocolMagic' pbft_genesis.json` --secret ../data/pbft0.key
2cWKMJemoBamE3NqsmfLZjLZ87jPzYLtByRQuvSngWjqKCJE3zh3T9Be43EU1PGQqon88
VerKey address with root f57cfc08dd0db9f2806a35a2fb3770a7cfaa08972f074c96b7c48fbf, attributes: AddrAttributes { derivation path: {} }

```
