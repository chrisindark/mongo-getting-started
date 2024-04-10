# mongo-getting-started

# setup mongo server on mac
brew tap mongodb/brew
brew update
brew install mongodb-community@6.0

brew services list
brew services start mongodb-community

// connect to mongod server from shell using mongosh
mongosh

// for monitoring, import and export
mongotop
mongoimport
mongodump

## setup of mongo server
use admin;

// create admin user for new mongo server
db.createUser({
    user: "adminUser",
    pwd: "adminPassword",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
})

// create custom user for applications to use
db.createUser({
    user: "myUserAdmin",
    pwd: "myUserAdmin",
    roles: [
        { role: "readWrite", db: "test" }
    ]
})

## Sharding
- Set up a Sharded Cluster: Begin by setting up a sharded cluster with a minimum of one shard. You can add more shards later as your data grows. To start a sharded cluster on your local machine for testing purposes, you need at least three MongoDB instances: one for the config server, one for the router (mongos), and one for the shard. You can run multiple shards on the same machine using different ports.

// command to get server config paths
db.serverCmdLineOpts()

// dbpath of mongo instance installed by brew
/opt/homebrew/var/mongodb

// conf path of mongo instance installed by brew
/opt/homebrew/etc/mongod.conf

// ubuntu prod example 
mongos --configdb config1.example.net:27019,config2.example.net:27019,config3.example.net:27019 --bind_ip localhost --port 27017

## Local setup in 4 different terminal sessions
// This is the MongoDB server daemon. It's responsible for managing data and handling client requests. When you start mongod, you're starting an instance of the MongoDB server.
// --configsvr: This option tells mongod that it's going to be part of a configuration server replica set. Configuration servers store metadata and configuration settings for the sharded cluster.
// --replSet: This option specifies the name of the replica set to which the mongod instance belongs. Replica sets provide redundancy and high availability by maintaining multiple copies of data across different servers.

// config servers should be minimum 3 in production
mongod --configsvr --replSet configReplSet --dbpath /opt/homebrew/var/mongodb-sharded-configserver-1 --port 27019
mongod --configsvr --replSet configReplSet --dbpath /opt/homebrew/var/mongodb-sharded-configserver-2 --port 27020
mongod --configsvr --replSet configReplSet --dbpath /opt/homebrew/var/mongodb-sharded-configserver-3 --port 27021

// This option tells mongod that it's going to be a shard server. Shard servers store data and are responsible for managing a portion of the data in the sharded cluster.
// --shardsvr: This option tells mongod that it's going to be a shard server. Shard servers store data and are responsible for managing a portion of the data in the sharded cluster.
// --replSet shardX: This option specifies the name of the replica set to which the mongod instance belongs. Each shard should be part of its own replica set. Replace shardX with a meaningful name for your shard, such as shard1 or shard2.

// shard servers should be minimum 2 in production
mongod --shardsvr --replSet shard1 --dbpath /opt/homebrew/var/mongodb-sharded-shard-1 --port 27018
// replicas for a shard server can be added as well:
mongod --shardsvr --replSet shard1 --dbpath /opt/homebrew/var/mongodb-sharded-shard-1-replica-1 --port 27015
mongod --shardsvr --replSet shard1 --dbpath /opt/homebrew/var/mongodb-sharded-shard-1-replica-2 --port 27014

mongod --shardsvr --replSet shard2 --dbpath /opt/homebrew/var/mongodb-sharded-shard-2 --port 27016
// replicas for a shard server can be added as well:
mongod --shardsvr --replSet shard2 --dbpath /opt/homebrew/var/mongodb-sharded-shard-2-replica-1 --port 27013
mongod --shardsvr --replSet shard2 --dbpath /opt/homebrew/var/mongodb-sharded-shard-2-replica-2 --port 27012

// This is the MongoDB shell for sharded clusters, also known as the MongoDB router process. It's responsible for routing queries and write operations to the appropriate shard(s) in the sharded cluster.
// --configdb configReplSet/localhost:27019: This option specifies the configuration servers that mongos should connect to. In this case, configReplSet is the name of the replica set for the config servers, and localhost:27019 is the address of one of the config servers. If you have multiple config servers, you would provide a comma-separated list of their addresses here.
mongos --configdb <config-server-addresses> --bind_ip <mongos-ip-address> --port <mongos-port>
mongos --configdb configReplSet/localhost:27019,localhost:27020,localhost:27021 --port 27017

// connect to one of the config servers and initialize the config replica set:
mongosh --port 27019

rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host : "localhost:27019" },
    <!-- { _id: 1, host : "localhost:27020" }, -->
    <!-- { _id: 2, host : "localhost:27021" } -->
  ]
})

// connect to one of the shard servers and initialize the config replica set
mongosh --port 27018

rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host : "localhost:27018" }
    <!-- { _id: 1, host : "localhost:27015" }, -->
    <!-- { _id: 2, host : "localhost:27014" }, -->
  ]
})

rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host : "localhost:27016" },
    <!-- { _id: 0, host : "localhost:27013" }, -->
    <!-- { _id: 0, host : "localhost:27012" }, -->
  ]
})

// to reconfig the replicaset configuration use this:
rs.reconfig({ _id: "shard1", members: [{ _id: 0, host : "localhost:27018" }] })
rs.reconfig({ _id: "shard2", members: [{ _id: 0, host : "localhost:27016" }] })


// connect to the mongos instance/s and add the shard servers to them using the sh.addShard() command.
mongosh --port 27017

sh.addShard("shard1/localhost:27018")
sh.addShard("shard2/localhost:27016")
<!-- sh.addShard("shard3/localhost:27015") -->

sh.status() output:
shardingVersion
{ _id: 1, clusterId: ObjectId('6615bb9b4ca6b2d974f31927') }
---
shards
[
  {
    _id: 'shard1',
    host: 'shard1/localhost:27018',
    state: 1,
    topologyTime: Timestamp({ t: 1712702743, i: 4 })
  },
  {
    _id: 'shard2',
    host: 'shard2/localhost:27016',
    state: 1,
    topologyTime: Timestamp({ t: 1712703368, i: 4 })
  }
]
---

// NOT BE NEEDED IN NEW MONGO VERSIONS
// mongosh to the primary shard server, switch to admin database, and add a shard identity for the shard server
mongosh --port 27016

use admin;
db.system.version.find();

db.system.version.insert({
  "_id" : "shardIdentity",
  "clusterId" : ObjectId("6615bb9b4ca6b2d974f31927"),
  "shardName" : "shard2",
  "configsvrConnectionString" : "configReplSet/localhost:27019"
})

// switch to the other shard servers, switch to admin and check if the identities are generated or not
mongosh --port 27018

use admin;
db.system.version.find();

db.system.version.insert({
  "_id" : "shardIdentity",
  "clusterId" : ObjectId("6615bb9b4ca6b2d974f31927"),
  "shardName" : "shard1",
  "configsvrConnectionString" : "configReplSet/localhost:27019"
})

// Finally, enable sharding for databases and collections as needed using the sh.enableSharding() and sh.shardCollection() commands.
// Enable sharding for a database
<!-- sh.enableSharding("myDatabase") -->
sh.enableSharding("test")

// Shard a collection
// Range-based sharding can use multiple fields as the shard key and divides data into contiguous ranges determined by the shard key values.
<!-- sh.shardCollection("myDatabase.myCollection", { "shardKey": 1, ... }) -->
// Hashed sharding uses a hashed index of a single field as the shard key to partition data across your sharded cluster.
sh.shardCollection("test.cats", { _id: "hashed" })
