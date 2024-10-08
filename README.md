
# MongoDB Sharding - Single Node Macbook - Quick Setup

There are times when I need to quickly set up a shard, or even a MongoDB cluster with 2 or 3 shards. This guide makes it simple and configurable to do so with a single command. No Docker is involved—just plain old MongoDB, specifically version 8, which is the latest at the time of writing.

## Overview

Let's begin by installing all the necessary binaries.

- **mongosh**: Download the [***MongoDB shell***](https://www.mongodb.com/try/download/shell), as we'll need this multiple times throughout the setup.
- **mongod**: Grab the latest [***MongoDB server***](https://www.mongodb.com/try/download/enterprise), required for each instance.

After downloading, unzip the contents and move the `bin` folder contents to your system’s environment paths.

You should now be able to run the following commands to verify the versions of both.

```text
mongosh --version
2.3.1

mongod --version
db version v8.0.0
Build Info: {
    "version": "8.0.0",
    "gitVersion": "d7cd03b239ac39a3c7d63f7145e91aca36f93db6",
    "modules": [
        "enterprise"
    ],
    "allocator": "system",
    "environment": {
        "distarch": "x86_64",
        "target_arch": "x86_64"
    }
}
mongos --version
mongos version v8.0.0
Build Info: {
    "version": "8.0.0",
    "gitVersion": "d7cd03b239ac39a3c7d63f7145e91aca36f93db6",
    "modules": [
        "enterprise"
    ],
    "allocator": "system",
    "environment": {
        "distarch": "x86_64",
        "target_arch": "x86_64"
    }
}
```

Once everything is confirmed, we're good to go. We'll follow the sequence below:

- **Replica Sets**: First, we'll create a simple replica set and then tear it down.
- **Shards**: Set up a sharded cluster with 2 shards.
- **Data**: Insert some sample data.
- **Collections**: Shard the collection.
- **Config Shard**: Convert the config server(s) into a data shard.
- **Wrap up**: What’s next?


### Replica Set

Start by creating a file named `mongo-replicaset-create` to set up the replica set.

```shell
NAME=$1
PORT=$2
DIR=~/data/$NAME
NODES=$3
USAGE="\nUsage:  mongo-replicaset-create {name} {port} {numNodes} { Optional:[ --shardsvr, --configsvr ] }\nSet env variable \$HOST for all nodes to use this domain name."
SLEEPTIME=8
CONFIGSVR=""
SHARDSVR=""
GIVEN_HOSTNAME=localhost

if [ -z "$HOST" ]
then
      echo "\$HOST not set using localhost"
else
      GIVEN_HOSTNAME=$HOST
fi

if [ "$1" == "-h" ]; then
  echo -e $USAGE
  exit 0
fi

if [ -z "$1" ]
  then
    echo -e "No NAME argument supplied $USAGE"
    exit 0
fi

if [ -z "$2" ]
  then
    echo -e "No PORT argument supplied $USAGE"
    exit 0
fi

if [ -z "$3" ]
  then
    echo -e "Number of NODES  argument supplied $USAGE"
    exit 0
fi

# idiomatic parameter and option handling in sh
while test $# -gt 0
do
    case "$4" in
        --shardsvr) echo "Using option --shardsvr"
	    SHARDSVR="--shardsvr"
            ;;
        --configsvr) echo "Using option --configsvr"
	    CONFIGSVR="--configsvr"
            ;;
        --*) echo "bad option $1"
            ;;
    esac
    shift
done

rm -fr $DIR
mkdir -p $DIR

COUNTER=0
  until [  $COUNTER -ge $NODES ]; do
    LOC=$DIR/$COUNTER
    mkdir -p $LOC
    PC=$(($PORT+$COUNTER))
    echo "[ ++ Executing:  mongod --port $PC --fork --dbpath $LOC --logpath $LOC/log.out -replSet $NAME $CONFIGSVR $SHARDSVR ]"
    mongod --port $PC --fork --bind_ip localhost,$GIVEN_HOSTNAME --dbpath $LOC --logpath $LOC/log.out --replSet $NAME $CONFIGSVR $SHARDSVR
    let COUNTER+=1
  done

echo "Sleeping for $SLEEPTIME seconds"
sleep $SLEEPTIME

echo "[ ++ Executing:  mongosh --port $PORT -eval \"rs.initiate()\" ]"
mongosh --port $PORT -eval "rs.initiate()"
sleep $SLEEPTIME

COUNTER=1
  until [  $COUNTER -ge $NODES ]; do
    PC=$(($PORT+$COUNTER))
    if [ $COUNTER -gt 6 ]; then
	echo "[ ++ Executing: mongosh --port $PORT -eval \"rs.add({host: '$GIVEN_HOSTNAME:$PC', priority:0, votes:0})\" ]"
	mongosh --port $PORT -eval "rs.add({host: '$GIVEN_HOSTNAME:$PC', priority:0, votes:0})"
    else
    	echo "[ ++ Executing:  mongosh --port $PORT -eval \"rs.add('$GIVEN_HOSTNAME:$PC')\" ]"
    	mongosh --port $PORT -eval "rs.add('$GIVEN_HOSTNAME:$PC')"
    fi
    let COUNTER+=1
  done

sleep $SLEEPTIME
mongosh --port $PORT -eval "rs.status()"
```

Create the replica set and capture the stdout if you want to inspect the output later. It makes sense to set the `HOST` variable so that it can be used as a Fully Qualified Domain Name (FQDN) during the creation process.

```bash
# Set the host
export HOST=`hostname`

# Check we set it correctly
env | grep HOST
HOST=nmaharaj-MBP-MMD6T.local
```

Run the script

```bash
mongo-replicaset-create myrs 27017 3
```

Here is the result and output
```text
[ ++ Executing:  mongod --port 27017 --fork --dbpath /Users/nareshmaharaj/data/myrs/0 --logpath /Users/nareshmaharaj/data/myrs/0/log.out -replSet myrs   ]
about to fork child process, waiting until server is ready for connections.
forked process: 43633
child process started successfully, parent exiting
[ ++ Executing:  mongod --port 27018 --fork --dbpath /Users/nareshmaharaj/data/myrs/1 --logpath /Users/nareshmaharaj/data/myrs/1/log.out -replSet myrs   ]
about to fork child process, waiting until server is ready for connections.
forked process: 43664
child process started successfully, parent exiting
[ ++ Executing:  mongod --port 27019 --fork --dbpath /Users/nareshmaharaj/data/myrs/2 --logpath /Users/nareshmaharaj/data/myrs/2/log.out -replSet myrs   ]
about to fork child process, waiting until server is ready for connections.
forked process: 43689
child process started successfully, parent exiting
Sleeping for 8 seconds
[ ++ Executing:  mongosh --port 27017 -eval "rs.initiate()" ]
{
  info2: 'no configuration specified. Using a default configuration for the set',
  me: 'nmaharaj-MBP-MMD6T.local:27017',
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728146621, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728146621, i: 1 })
}
[ ++ Executing:  mongosh --port 27017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:27018')" ]
{
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728146631, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728146631, i: 1 })
}
[ ++ Executing:  mongosh --port 27017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:27019')" ]
{
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728146633, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728146633, i: 1 })
}
{
  set: 'myrs',
  date: ISODate('2024-10-05T16:44:02.989Z'),
  myState: 1,
  term: Long('1'),
  syncSourceHost: '',
  syncSourceId: -1,
  heartbeatIntervalMillis: Long('2000'),
  majorityVoteCount: 2,
  writeMajorityCount: 2,
  votingMembersCount: 3,
  writableVotingMembersCount: 3,
  optimes: {
    lastCommittedOpTime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
    lastCommittedWallTime: ISODate('2024-10-05T16:44:01.930Z'),
    readConcernMajorityOpTime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
    appliedOpTime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
    durableOpTime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
    writtenOpTime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
    lastAppliedWallTime: ISODate('2024-10-05T16:44:01.930Z'),
    lastDurableWallTime: ISODate('2024-10-05T16:44:01.930Z'),
    lastWrittenWallTime: ISODate('2024-10-05T16:44:01.930Z')
  },
  lastStableRecoveryTimestamp: Timestamp({ t: 1728146621, i: 1 }),
  electionCandidateMetrics: {
    lastElectionReason: 'electionTimeout',
    lastElectionDate: ISODate('2024-10-05T16:43:42.053Z'),
    electionTerm: Long('1'),
    lastCommittedOpTimeAtElection: { ts: Timestamp({ t: 1728146621, i: 1 }), t: Long('-1') },
    lastSeenWrittenOpTimeAtElection: { ts: Timestamp({ t: 1728146621, i: 1 }), t: Long('-1') },
    lastSeenOpTimeAtElection: { ts: Timestamp({ t: 1728146621, i: 1 }), t: Long('-1') },
    numVotesNeeded: 1,
    priorityAtElection: 1,
    electionTimeoutMillis: Long('10000'),
    newTermStartDate: ISODate('2024-10-05T16:43:42.465Z'),
    wMajorityWriteAvailabilityDate: ISODate('2024-10-05T16:43:42.735Z')
  },
  members: [
    {
      _id: 0,
      name: 'nmaharaj-MBP-MMD6T.local:27017',
      health: 1,
      state: 1,
      stateStr: 'PRIMARY',
      uptime: 40,
      optime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
      optimeDate: ISODate('2024-10-05T16:44:01.000Z'),
      optimeWritten: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
      optimeWrittenDate: ISODate('2024-10-05T16:44:01.000Z'),
      lastAppliedWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastDurableWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastWrittenWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: 'Could not find member to sync from',
      electionTime: Timestamp({ t: 1728146622, i: 1 }),
      electionDate: ISODate('2024-10-05T16:43:42.000Z'),
      configVersion: 5,
      configTerm: 1,
      self: true,
      lastHeartbeatMessage: ''
    },
    {
      _id: 1,
      name: 'nmaharaj-MBP-MMD6T.local:27018',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 11,
      optime: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
      optimeDurable: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
      optimeWritten: { ts: Timestamp({ t: 1728146641, i: 1 }), t: Long('1') },
      optimeDate: ISODate('2024-10-05T16:44:01.000Z'),
      optimeDurableDate: ISODate('2024-10-05T16:44:01.000Z'),
      optimeWrittenDate: ISODate('2024-10-05T16:44:01.000Z'),
      lastAppliedWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastDurableWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastWrittenWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastHeartbeat: ISODate('2024-10-05T16:44:02.075Z'),
      lastHeartbeatRecv: ISODate('2024-10-05T16:44:02.075Z'),
      pingMs: Long('9'),
      lastHeartbeatMessage: '',
      syncSourceHost: 'nmaharaj-MBP-MMD6T.local:27017',
      syncSourceId: 0,
      infoMessage: '',
      configVersion: 5,
      configTerm: 1
    },
    {
      _id: 2,
      name: 'nmaharaj-MBP-MMD6T.local:27019',
      health: 1,
      state: 2,
      stateStr: 'SECONDARY',
      uptime: 9,
      optime: { ts: Timestamp({ t: 1728146637, i: 1 }), t: Long('1') },
      optimeDurable: { ts: Timestamp({ t: 1728146637, i: 1 }), t: Long('1') },
      optimeWritten: { ts: Timestamp({ t: 1728146637, i: 1 }), t: Long('1') },
      optimeDate: ISODate('2024-10-05T16:43:57.000Z'),
      optimeDurableDate: ISODate('2024-10-05T16:43:57.000Z'),
      optimeWrittenDate: ISODate('2024-10-05T16:43:57.000Z'),
      lastAppliedWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastDurableWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastWrittenWallTime: ISODate('2024-10-05T16:44:01.930Z'),
      lastHeartbeat: ISODate('2024-10-05T16:44:02.078Z'),
      lastHeartbeatRecv: ISODate('2024-10-05T16:44:02.579Z'),
      pingMs: Long('3'),
      lastHeartbeatMessage: '',
      syncSourceHost: '',
      syncSourceId: -1,
      infoMessage: '',
      configVersion: 5,
      configTerm: 1
    }
  ],
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728146641, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728146641, i: 1 })
}
```

All the commands that were run are shown with the prefix: ++ Executing: ...

Next, log in to the host.


```text
mongosh nmaharaj-MBP-MMD6T.local:27019
```
Now, tear everything down by running the following commands. Make sure to wait for all processes to be completely removed.

```bash
kill `ps -elf | grep mongo | awk '{print $2}' | tr '\n' ' ' | awk '{$1=$1};1'` > /dev/null 2>&1
```

### Sharded Cluster

To create a sharded cluster with a config server replicaset and a mongos router, create a file named `mongo-shard-create` with the following contents.


```bash
PORT=$1
SHARDS=$2
LISTNAMES=$3
DIR=~/data/mongos/
USAGE="Usage:  mongo-shard-create {port} {numShards} { shardName1,shardName2,..}"
GIVEN_HOSTNAME=localhost
SLEEPTIME=1

if [ -z "$HOST" ]
then
      echo "\$HOST is empty but is optional - will use localhost"
else
      GIVEN_HOSTNAME=$HOST
fi


if [ -z "$1" ]
  then
    echo "No PORT argument supplied - $USAGE"
    exit 0
fi

if [ -z "$2" ]
  then
    echo "Number of Shards required - $USAGE"
    exit 0
fi

if [ -z "$3" ]
  then
    echo "Name of each shard csv required - $USAGE"
    exit 0
fi

# Split the names for each RS
rs_name_arr=($(echo "$3" | sed "s/ //g" | tr ',' '\n'))
count=${#rs_name_arr[@]}
if [ $count -lt $SHARDS ]
  then
    echo "Shard names(s) count does not match number of Shards required - $USAGE"
    exit 0
fi

echo "##########################################################################################################"
echo "######################################### Creating Replica Sets ##########################################"
echo "##########################################################################################################"
COUNTER=1
  until [  $COUNTER -gt $SHARDS ]; do
  name=${rs_name_arr[$COUNTER-1]}
    PC=$(($PORT+17+1000*$COUNTER))
    mongo-replicaset-create $name $PC 3 --shardsvr
    let COUNTER+=1
  done

echo "##########################################################################################################"
echo "######################################### Creating Config Sets ###########################################"
echo "##########################################################################################################"
# Config Servers
PC=$(($PORT+17+$SHARDS*1000+1000))
mongo-replicaset-create csrs $PC 3 --configsvr

echo " *** Adding user: 'admin', pwd: 'password' to Config Server - please change password *** "
CSRS_HOSTS="csrs/$GIVEN_HOSTNAME:$PC,$GIVEN_HOSTNAME:$(($PC+1)),$GIVEN_HOSTNAME:$(($PC+2))"
mongosh --host $CSRS_HOSTS --eval "db.getSiblingDB('admin').createUser({ user: 'admin', pwd: 'password', roles: [ {role: 'root', db: 'admin'} ] })"


echo "##########################################################################################################"
echo "######################################### Creating mongos #### ###########################################"
echo "##########################################################################################################"
LOG_DIR_MONGOS=~/data/mongos/
mkdir -p $LOG_DIR_MONGOS
echo "[ ++ Executing:  mongos --port $(($PORT+17)) --bind_ip_all --fork --logpath $LOG_DIR_MONGOS/mongos.log --configdb $CSRS_HOSTS ]"
mongos --port $(($PORT+17)) --bind_ip_all --fork --logpath $LOG_DIR_MONGOS/mongos.log --configdb $CSRS_HOSTS

COUNTER=1
  until [  $COUNTER -gt $SHARDS ]; do
  name=${rs_name_arr[$COUNTER-1]}
    PC=$(($PORT+17+1000*$COUNTER))
    echo "[ ++ Executing: mongosh --port $(($PORT+17)) --eval \"sh.addShard('$name/$GIVEN_HOSTNAME:$PC')\" ]"
    mongosh --port $(($PORT+17)) --eval "sh.addShard('$name/$GIVEN_HOSTNAME:$PC')"
    let COUNTER+=1
  done

sleep 5
mongosh --port $(($PORT+17)) --eval "sh.status()"
```

Run the following command to create a MongoDB sharded cluster with 2 shards.

```bash
mongo-shard-create 27017 2 sh1,sh2 | tee ~/out.log
```

Here is the output; however, we have omitted most of it due to its large size. We have included all the executed commands and the last part of the stdout.

```text
# commands executed
cat ~/out.log | grep -e Exec -e pass
[ ++ Executing:  mongod --port 28017 --fork --dbpath /Users/nareshmaharaj/data/sh1/0 --logpath /Users/nareshmaharaj/data/sh1/0/log.out -replSet sh1  --shardsvr ]
[ ++ Executing:  mongod --port 28018 --fork --dbpath /Users/nareshmaharaj/data/sh1/1 --logpath /Users/nareshmaharaj/data/sh1/1/log.out -replSet sh1  --shardsvr ]
[ ++ Executing:  mongod --port 28019 --fork --dbpath /Users/nareshmaharaj/data/sh1/2 --logpath /Users/nareshmaharaj/data/sh1/2/log.out -replSet sh1  --shardsvr ]
[ ++ Executing:  mongosh --port 28017 -eval "rs.initiate()" ]
[ ++ Executing:  mongosh --port 28017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:28018')" ]
[ ++ Executing:  mongosh --port 28017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:28019')" ]
[ ++ Executing:  mongod --port 29017 --fork --dbpath /Users/nareshmaharaj/data/sh2/0 --logpath /Users/nareshmaharaj/data/sh2/0/log.out -replSet sh2  --shardsvr ]
[ ++ Executing:  mongod --port 29018 --fork --dbpath /Users/nareshmaharaj/data/sh2/1 --logpath /Users/nareshmaharaj/data/sh2/1/log.out -replSet sh2  --shardsvr ]
[ ++ Executing:  mongod --port 29019 --fork --dbpath /Users/nareshmaharaj/data/sh2/2 --logpath /Users/nareshmaharaj/data/sh2/2/log.out -replSet sh2  --shardsvr ]
[ ++ Executing:  mongosh --port 29017 -eval "rs.initiate()" ]
[ ++ Executing:  mongosh --port 29017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:29018')" ]
[ ++ Executing:  mongosh --port 29017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:29019')" ]
[ ++ Executing:  mongod --port 30017 --fork --dbpath /Users/nareshmaharaj/data/csrs/0 --logpath /Users/nareshmaharaj/data/csrs/0/log.out -replSet csrs --configsvr  ]
[ ++ Executing:  mongod --port 30018 --fork --dbpath /Users/nareshmaharaj/data/csrs/1 --logpath /Users/nareshmaharaj/data/csrs/1/log.out -replSet csrs --configsvr  ]
[ ++ Executing:  mongod --port 30019 --fork --dbpath /Users/nareshmaharaj/data/csrs/2 --logpath /Users/nareshmaharaj/data/csrs/2/log.out -replSet csrs --configsvr  ]
[ ++ Executing:  mongosh --port 30017 -eval "rs.initiate()" ]
[ ++ Executing:  mongosh --port 30017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:30018')" ]
[ ++ Executing:  mongosh --port 30017 -eval "rs.add('nmaharaj-MBP-MMD6T.local:30019')" ]
 *** Adding user: 'admin', pwd: 'password' to Config Server - please change password ***
[ ++ Executing:  mongos --port 27017 --bind_ip_all --fork --logpath /Users/nareshmaharaj/data/mongos//mongos.log --configdb csrs/nmaharaj-MBP-MMD6T.local:30017,nmaharaj-MBP-MMD6T.local:30018,nmaharaj-MBP-MMD6T.local:30019 ]
[ ++ Executing: mongosh --port 27017 --eval "sh.addShard('sh1/nmaharaj-MBP-MMD6T.local:28017')" ]
[ ++ Executing: mongosh --port 27017 --eval "sh.addShard('sh2/nmaharaj-MBP-MMD6T.local:29017')" ]

# stdout
...
##########################################################################################################
######################################### Creating mongos #### ###########################################
##########################################################################################################
[ ++ Executing:  mongos --port 27017 --bind_ip_all --fork --logpath /Users/nareshmaharaj/data/mongos//mongos.log --configdb csrs/nmaharaj-MBP-MMD6T.local:30017,nmaharaj-MBP-MMD6T.local:30018,nmaharaj-MBP-MMD6T.local:30019 ]
about to fork child process, waiting until server is ready for connections.
forked process: 51366
child process started successfully, parent exiting
[ ++ Executing: mongosh --port 27017 --eval "sh.addShard('sh1/nmaharaj-MBP-MMD6T.local:28017')" ]
{
  shardAdded: 'sh1',
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728147532, i: 6 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728147532, i: 6 })
}
[ ++ Executing: mongosh --port 27017 --eval "sh.addShard('sh2/nmaharaj-MBP-MMD6T.local:29017')" ]
{
  shardAdded: 'sh2',
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728147535, i: 16 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728147535, i: 10 })
}
shardingVersion
{ _id: 1, clusterId: ObjectId('670170310cdeaa5de24f0045') }
---
shards
[
  {
    _id: 'sh1',
    host: 'sh1/nmaharaj-MBP-MMD6T.local:28017,nmaharaj-MBP-MMD6T.local:28018,nmaharaj-MBP-MMD6T.local:28019',
    state: 1,
    topologyTime: Timestamp({ t: 1728147531, i: 6 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'sh2',
    host: 'sh2/nmaharaj-MBP-MMD6T.local:29017,nmaharaj-MBP-MMD6T.local:29018,nmaharaj-MBP-MMD6T.local:29019',
    state: 1,
    topologyTime: Timestamp({ t: 1728147535, i: 1 }),
    replSetConfigVersion: Long('-1')
  }
]
---
active mongoses
[ { '8.0.0': 1 } ]
---
autosplit
{ 'Currently enabled': 'yes' }
---
balancer
{
  'Currently enabled': 'yes',
  'Currently running': 'no',
  'Failed balancer rounds in last 5 attempts': 0,
  'Migration Results for the last 24 hours': 'No recent migrations'
}
---
databases
[
  {
    database: { _id: 'config', primary: 'config', partitioned: true },
    collections: {}
  }
]
```

As you can see, we now have a fully configured sharded cluster, and it only takes about a minute to set up.

Log into mongos using `mongosh` on port 27017 and check the status of the shards.

```text
mongosh --port 27017 --eval "sh.status()"
```

Note that there are only 2 shards. As of version 8.0, you can transition the config servers to [data shards](https://www.mongodb.com/docs/manual/reference/command/transitionFromDedicatedConfigServer/#mongodb-dbcommand-dbcmd.transitionFromDedicatedConfigServer). Let's see how this is done.

Log in to mongos using `mongosh` on port 27017.

```text
mongosh --port 27017"
```

Run the following command:
```bash
db.adminCommand({transitionFromDedicatedConfigServer:1})
{
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1728148222, i: 12 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1728148222, i: 12 })
}
```

Now, check to see how many data shards are present.
```text
mongosh --port 27017 --eval "sh.status()"
```

You should now see all 3

```text
shards
[
  {
    _id: 'config',
    host: 'csrs/nmaharaj-MBP-MMD6T.local:30017,nmaharaj-MBP-MMD6T.local:30018,nmaharaj-MBP-MMD6T.local:30019',
    state: 1,
    topologyTime: Timestamp({ t: 1728148222, i: 3 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'sh1',
    host: 'sh1/nmaharaj-MBP-MMD6T.local:28017,nmaharaj-MBP-MMD6T.local:28018,nmaharaj-MBP-MMD6T.local:28019',
    state: 1,
    topologyTime: Timestamp({ t: 1728147531, i: 6 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'sh2',
    host: 'sh2/nmaharaj-MBP-MMD6T.local:29017,nmaharaj-MBP-MMD6T.local:29018,nmaharaj-MBP-MMD6T.local:29019',
    state: 1,
    topologyTime: Timestamp({ t: 1728147535, i: 1 }),
    replSetConfigVersion: Long('-1')
  }
]
```
