## Create new docker image for BigchainDB

Following:
* https://www.jamescoyle.net/how-to/1503-create-your-first-docker-container
* http://docs.bigchaindb.com/projects/server/en/latest/simple-deployment-template/deploy-a-machine.html

We want to use port 9984 with HTTP for local BigchainDB testing node.

Download base OS image to start with:

```
$ docker pull ubuntu:18.04
```

Install BigchainDB inside the base image. First update the OS.

```
$ docker images | grep ubuntu # Look up the image ID (e.g. 94e814e2efa8)
$ docker run -i -t 94e814e2efa8 /bin/bash
  $ apt update
  $ apt full-upgrade
```

Note: `docker run` creates a new container with new `CONTAINER_ID` from the image.
Afterwards, you can start and stop the container as you like.

Note: You can exit the container with `exit` (stops the container, can also do
`docker stop CONTAINER_ID` from outside) and then enter it again with
`docker start CONTAINER_ID` and `docker attach CONTAINER_ID`. Find
`CONTAINER_ID` with `docker ps -a`.

You can detach from the container and leave it running with CTRL-p CTRL-q.

Skip the steps about NGINX (needed for production, but not for dev).

Install python3 and pip:

```
  $ apt install -y python3-pip libssl-dev
```

Look up most recent version of bigchaindb in python package index on https://pypi.org/project/BigchainDB/#history, e.g. 2.0.0b9.
Install that version with pip:

```
  $ pip3 install BigchainDB==2.0.0b9
# Check that it worked with
  $ bigchaindb --version
# Configure:
  $ bigchaindb configure
# On first question (API Server bind?) enter `0.0.0.0:9984` (because not using NGINX)
# Accept the default value for all other BigchainDB config settings
```

Install MongoDB:

```
  $ apt install mongodb
```

Install Tendermint. For this version of BigchainDB, must use Tendermint version 0.22.8:

```
  $ apt install -y wget unzip
  $ cd # (Go to home directory, i.e. /root)
  $ wget https://github.com/tendermint/tendermint/releases/download/v0.22.8/tendermint_0.22.8_linux_amd64.zip
  $ unzip tendermint_0.22.8_linux_amd64.zip
  $ rm tendermint_0.22.8_linux_amd64.zip
  $ mv tendermint /usr/local/bin
```

Install Monit:

```
  $ apt install monit
```

Install netcat, netstat (in net-tools) and ping (in iputils-ping) for network testing purposes:

```
  $ apt install netcat net-tools iputils-ping
```

Create script to later easily start the bigchain network:

```
  $ echo '#!/bin/bash
mongod --fork --logpath /var/log/mongodb/mongodb.log
monit -d 1
monit summary' > /root/start_bigchain.sh
  $ chmod +x /root/start_bigchain.sh
```

Save your modifications as new Docker image:

```
  $ exit
$ docker ps -a # Look up container ID in the first row (or look at the docker root shell that started with root@...)
# Use that ID in the next commands:
$ docker start c5857c032765
$ docker attach c5857c032765
  $ which bigchaindb # check if it's still there (you could also have run all install commands inside this attach shell)
  $ exit
$ docker commit c5857c032765 bigchaindb-dev-node
```

In the last command, you can choose a name for the image (here:
bigchaindb-dev-node).

Run `docker ps -a` to look at list of once started, stil present containers.
Remove unneeded containers with `docker rm CONTAINER_ID`.

Running `docker system prune` also clears old volumes.

Run `docker images` to look at list of downloaded/created images.
Remove unneded images with `docker rmi IMAGE_ID`.


## Start up four BigchainDB nodes to create a local test network

### Create the test network

```
$ docker network create --subnet=172.18.0.0/16 bigchaindb-network
```

The containers run with `--net bigchaindb-network` will be able to communicate
with each other.

Check with

```
$ docker network ls
```

### Start the coordinator node

------------------------------

Didn't work out:

~~Start the first instance of BigchainDB (a container). E.g. as a daemon running in background: (https://linuxconfig.org/how-to-start-a-docker-container-as-daemon-process)~~

```
~~$ docker run --net bigchaindb-network --ip 172.18.0.20 --name bigchaindb-coordinator -p 39984:9984 -d bigchaindb-dev-node /bin/sh -c "while true; do date; sleep 5; done"~~
```

~~Alternative: use an interactive (-it) session that you never exit:~~

```
~~$ docker run --net bigchaindb-network --ip 172.18.0.20 --name bigchaindb-coordinator -p 39984:9984 -it bigchaindb-dev-node /bin/bash~~
```

~~Local port 39984 is bound to the node port 9984 (the default BigchainDB port).~~
~~You can later reach the BigchainDB HTTP API on that port.~~

------------------------------

Instead, just do:

```
$ docker run --net bigchaindb-network --ip 172.18.0.20 --name bigchaindb-coordinator -it bigchaindb-dev-node /bin/bash
```

#### Optional: some docker commands

Look at `CONTAINER_ID` with:

```
$ docker ps
```

Look at logs using `CONTAINER_ID`:

```
$ docker logs 6acfc613c604
```

Show container's IP address: (https://stackoverflow.com/questions/17157721/how-to-get-a-docker-containers-ip-address-from-the-host)

```
# Does not work: $ docker exec -it bigchaindb-coordinator ip addr show
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' bigchaindb-coordinator
# Or:
$ docker inspect bigchaindb-coordinator | grep IPAddress
```

Attach to the running container with:

```
$ docker exec -it bigchaindb-coordinator /bin/bash
```

Stop the container:

```
$ docker stop CONTAINER_ID
```

Get the ID with `docker ps` (maybe one can use name instead of ID as well?).


### Start the member nodes

Run these commands each in a new terminal window:

```
$ docker run --net bigchaindb-network --ip 172.18.0.21 --name bigchaindb-member1 -it bigchaindb-dev-node /bin/bash
$ docker run --net bigchaindb-network --ip 172.18.0.22 --name bigchaindb-member2 -it bigchaindb-dev-node /bin/bash
$ docker run --net bigchaindb-network --ip 172.18.0.23 --name bigchaindb-member3 -it bigchaindb-dev-node /bin/bash
```

From each container, you should be able to reach the other containers. Test
with ping:

```
coordinator$ ping bigchaindb-member1
###
    member1$ ping bigchaindb-coordinator
###
...
```

or with netcat:

```
coordinator$ nc -lp 9984
###
     member$ nc bigchaindb-coordinator 9984
Hallo
^C
```


### Configure the nodes

On each node, both coordinator and members (in their respective terminal
window), generate node key and genesis file:

```
$ tendermint init
$ cat $HOME/.tendermint/config/priv_validator.json
$ tendermint show_node_id
```

Write down each node's public key (`pub_key.value`).
Write down each node's node ID.


#### Create the `genesis.json`

On the coordinator, edit the `genesis.json` file. For example:

```
coordinator$ cat $HOME/.tendermint/config/genesis.json
```

Copy the content and edit it in your favorite text editor. Add an entry in the
'validators' list for each node (with the node's public key). The file may look
like this: (`genesis.json`)

```
{
  "genesis_time": "2019-04-19T16:05:29.582811104Z",
  "chain_id": "test-chain-Xqw6gE",
  "consensus_params": {
    "block_size_params": {
      "max_bytes": "22020096",
      "max_txs": "10000",
      "max_gas": "-1"
    },
    "tx_size_params": {
      "max_bytes": "10240",
      "max_gas": "-1"
    },
    "block_gossip_params": {
      "block_part_size_bytes": "65536"
    },
    "evidence_params": {
      "max_age": "100000"
    }
  },
  "validators": [
    {
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "mHMLJtFS70uGSzghXLn7vjzhKW6nfiaVCAy28VPg/IQ="
      },
      "power": "10",
      "name": "coordinator"
    },
    {
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "T8qAw7EMMTm77C41IynbT86nHNIh2FZHOgMcfxMwLwg="
      },
      "power": "10",
      "name": "member1"
    },
    {
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "CCSg+U5oI0fLY7vkgybirWtft2scjNnm2Lkzv2TOak4="
      },
      "power": "10",
      "name": "member2"
    },
    {
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "8BQsXrHMgjvPGKYEX64fwIR4PKxzIVzbN62A7OgWs6E="
      },
      "power": "10",
      "name": "member3"
    }
  ],
  "app_hash": ""
}
```

Install this file on each node under `$HOME/.tendermint/config/genesis.json`.
E.g. copy all lines within the outer '{}' (without the lines with '{' and '}')
into the clipboard, then:

```
coordinator$ echo '{<ENTER>
<CTRL-SHIFT-V>
}' > $HOME/.tendermint/config/genesis.json<ENTER>
```

The `genesis.json`, especially the `genesis_time`, `chain_id` and list of
`validators`, must be identical on each node.

#### Edit each node's `config.toml`

```
$ cat $HOME/.tendermint/config/config.toml
```

Copy the content into your favorite text editor and make the following changes:

On each node, change:

```
create_empty_blocks = false
# ...
send_rate = 102400000
# ...
recv_rate = 102400000
# ...
recheck = false
```

On each node differently, set the moniker to the one you chose in the `genesis.json`:

```
moniker = "coordinator"
# Or:
moniker = "member1"
# Or:
...
```

On each node differently, set the connections to the other nodes by the scheme
`<tendermint_node_id>@<hostname>:26656`. For example, on the coordinator:

```
persistent_peers = "fc26e371a69cce6a5565b754c9593c1d31c961a7@bigchaindb-member1:26656,\
4155ac7583e4c6085e4b5090c57b875e672c7a57@bigchaindb-member2:26656,\
772e547610a95bcca1fd3bafd43b123defc8a264@bigchaindb-member3:26656,"
```

Or on member 1:

```
persistent_peers = "9e08ca6731dc924b9d89dc00048abbafb0567a01@bigchaindb-coordinator:26656,\
4155ac7583e4c6085e4b5090c57b875e672c7a57@bigchaindb-member2:26656,\
772e547610a95bcca1fd3bafd43b123defc8a264@bigchaindb-member3:26656,"
```

Install the `config.toml` files on the nodes, e.g. again with

```
$ echo '<CTRL-SHIFT-V>
' > $ HOME/.tendermint/config/config.toml
```

You can check if all is correct with:

```
$ grep moniker $HOME/.tendermint/config/config.toml
$ grep -3 persistent_peers $HOME/.tendermint/config/config.toml
```


#### Start MongoDB

On each node:

```
mkdir -p /data/db
mongod --fork --logpath /var/log/mongodb/mongodb.log
```


#### Start BigchainDB and Tendermint using Monit

On each node, create monit configuration file:

```
bigchaindb-monit-config
```

I think this creates the file `$HOME/.monitrc`.

Actually start up BigchainDB and Tendermint with:

```
$ monit -d 1
```

Monitor health of node with:

```
$ monit status
$ monit summary
```


### Access the BigchainDB HTTP API

Try if you can access BigchainDB at http://172.18.0.20:9984/ or http://172.18.0.21:9984/ or http://172.18.0.22:9984/ or http://172.18.0.23:9984/.


### Stop the BigchainDB network

In each container, run:

```
$ monit stop all
$ ps -ef | grep mongodb | head -1 # Look up PID (2nd column)
$ kill <PID>
```

Perhaps also need to:

```
$ ps -ef | grep bigchaindb | head -1 # Look up PID (2nd column)
$ kill <PID>
$ ps -ef | grep tendermint | head -1 # Look up PID (2nd column)
$ kill <PID>
```

Log out everywhere (CTRL-D or `exit`).


### Start the BigchainDB network again

In separate terminal windows:

```
$ docker start bigchaindb-coordinator
$ docker attach bigchaindb-coordinator
$ mongod --fork --logpath /var/log/mongodb/mongodb.log
$ monit -d 1
# Optional: check with
$ monit summary
```

```
$ docker start bigchaindb-member1
$ docker attach bigchaindb-member1
$ mongod --fork --logpath /var/log/mongodb/mongodb.log
$ monit -d 1
# Optional: check with
$ monit summary
```

```
$ docker start bigchaindb-member2
$ docker attach bigchaindb-member2
$ mongod --fork --logpath /var/log/mongodb/mongodb.log
$ monit -d 1
# Optional: check with
$ monit summary
```

```
$ docker start bigchaindb-member3
$ docker attach bigchaindb-member3
$ mongod --fork --logpath /var/log/mongodb/mongodb.log
$ monit -d 1
# Optional: check with
$ monit summary
```

