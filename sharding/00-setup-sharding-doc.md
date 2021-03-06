# Sharding with MongoDB
reference : Just me and Opensource

The Mongos are the interfaces to the shards. The application never interacts directly with the charts but with the gateway which are the mongos (lightweight) in this case. Then we have the config servers (as replica sets, needs to have at least two as if one is down, the mongos can still have the configuration information about the chards)  which store all the metadata and finally the shards which store the actual data. 

Best case for the configuration server is to have a replica sets of 3 servers, one primary and 2 secondary. Each replica set (server and the 2 shards) must have a different


## Set up Sharding using Docker Containers

### Config servers
Start config servers (3 member replica set)
```
docker-compose -f config-server/docker-compose.yaml up -d
```
Initiate replica set
```
mongo mongodb://<ip_address>:40001
```
It doesn't matter which config instance you log into (based on the port you choose above) but the one you choose to apply the following command to, will be the primary config server
```
rs.initiate(
  {
    _id: "cfgrs",
    configsvr: true,
    members: [
      { _id : 0, host : "<ip_address>:40001" },
      { _id : 1, host : "<ip_address>:40002" },
      { _id : 2, host : "<ip_address>:40003" }
    ]
  }
)

rs.status()
```

This result of the replica set will show us that indeed three members have been added to the replica set. One primary and two secondary. The two secondary servers will be syncing to the primary. 

### Shard 1 servers
Start shard 1 servers (3 member replicas set)
```
docker-compose -f shard1/docker-compose.yaml up -d
```
Initiate replica set
```
mongo mongodb://<ip_address>:50001
```
```
rs.initiate(
  {
    _id: "shard1rs",
    members: [
      { _id : 0, host : "<ip_address>:50001" },
      { _id : 1, host : "<ip_address>:50002" },
      { _id : 2, host : "<ip_address>:50003" }
    ]
  }
)

rs.status()
```

### Mongos Router
Start mongos query router
Before being able to run this command, you will need to change your ip address in the .env file in the sharding folder. example : IP_ADDRESS=<ip_address>
```
docker-compose -f mongos/docker-compose.yaml up -d
```

### Add shard to the cluster
Connect to mongos
```
mongo mongodb://<ip_address>:60000
```
Add shard
```
mongos> sh.addShard("shard1rs/<ip_address>:50001,<ip_address>:50002,<ip_address>:50003")
mongos> sh.status()
```
## Adding another shard
### Shard 2 servers
Start shard 2 servers (3 member replicas set)
```
docker-compose -f shard2/docker-compose.yaml up -d
```
Initiate replica set
```
mongo mongodb://<ip_address>:50004
```
```
rs.initiate(
  {
    _id: "shard2rs",
    members: [
      { _id : 0, host : "<ip_address>:50004" },
      { _id : 1, host : "<ip_address>:50005" },
      { _id : 2, host : "<ip_address>:50006" }
    ]
  }
)

rs.status()
```
### Add shard to the cluster
Connect to mongos
```
mongo mongodb://<ip_address>:60000
```
Add shard
```
mongos> sh.addShard("shard2rs/<ip_address>:50004,<ip_address>:50005,<ip_address>:50006")
mongos> sh.status()
```


# Sharding with mongoDB collections

First of, you will need to create collections in your databases. Indeed, we don't shard with dbs but with collections created. Thus to create a collection : 
```
db.createCollection("movies")
db.createCollection("books")
```

Now if you try to get the shard distributions of one of those collections 
```
db.movies.getShardDistribution()

>>> Collection sharddemo.movies is not Sharded.
```

To enable sharding (not working)
```
sh.shardCollection("sharddemo.movies", {"title": "hashed"})
```

In here, we have the collection we want to shard and then the shard key (to identify your data on the different shards) - multiple ways for the shard key (hashed)

```
OUTPUT
{
        "ok" : 0,
        "errmsg" : "sharding not enabled for db sharddemo",
        "code" : 20,
        "codeName" : "IllegalOperation",
        "operationTime" : Timestamp(1606230706, 5),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1606230706, 5),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
```
The problem here occurs as we didn't enable sharding in the database -> sh.enableSharding("sharddemo")
Now that we have enabled sharding in the database, we will be able to shard some collections. Sharded collection and non sharded collections can coexist in a database. 

Now if we try to enable sharding for the collection again, it should work. Here is an example output: 

```
>>> OUTPUT
{
        "collectionsharded" : "sharddemo.movies",
        "collectionUUID" : UUID("03f5061b-3465-4ff2-a8e8-0017ac1d12f5"),
        "ok" : 1,
        "operationTime" : Timestamp(1606231146, 13),
        "$clusterTime" : {
                "clusterTime" : Timestamp(1606231146, 13),
                "signature" : {
                        "hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
                        "keyId" : NumberLong(0)
                }
        }
}
```
And now if we try to get the shard distribution of the collection and


```
db.movies.getShardDistribution()
OUTPUT
>>> 

Shard shard1rs at shard1rs/<ip_address>:50001,<ip_address>:50002,<ip_address>:50003
 data : 0B docs : 0 chunks : 2
 estimated data per chunk : 0B
 estimated docs per chunk : 0

Shard shard2rs at shard2rs/<ip_address>:50004,<ip_address>:50005,<ip_address>:50006
 data : 0B docs : 0 chunks : 2
 estimated data per chunk : 0B
 estimated docs per chunk : 0

Totals
 data : 0B docs : 0 chunks : 4
 Shard shard1rs contains 0% data, 0% docs in cluster, avg obj size on shard : 0B
 Shard shard2rs contains 0% data, 0% docs in cluster, avg obj size on shard : 0B

```

db.books.getShardDistribution() will output Collection sharddemo.books is not Sharded. -> They can indeed coexist.

## TEST to populate databases
A simple test to populate the database can be found in the code at pymongo/mongodb.py. If the database and collections don't exist, these will be created. If you want one of the collections to be sharded, you will need to enable sharding manually with the instructions given at Sharding with mongoDB collections

To execute the different cruds method, simply run the whole python file. This will insert multiple elements in the database. Execute a query to find a certain element by its title in the collection and finally delete an element by its title in the given collection.

### test on the two types of collections, sharded and normal.

For the sharded collections, here are some of the experiments we have done. 

- if the primary shard is down (simply stop the docker container for one of the shard systems), retrieving the data will not be possible.
- if one secondary shard is down,, retrieving the data will be possible. However, if both of the secondary charts are down, the sharded collection will not be accessible. 

For the normal collection: 

Before explaining what we have done, we first have to look what the primary shard is of our database. In our case, the database sharddemo has the following as configurations.

´´´
{ 
  id: "shardddemo"
  primary: "shard2rs"
  partitioned: true
}
´´´

As we can see, the primary instance of our database is in the second shard. Now here is what we have discovered:
- if the shard1 instance is partially down, completely down or completely up, it doesn't matter for the collection.
- if the shard2 instance has one of the secondary down, the collection is still accessible. However if the primary is down, or all secondary are down, the database will not be accessible.


One more thing, if the configuration servers are down, the database is still accessible. 

TODO
- find pymongo method to modify an element in the database
- find out how the shutdown of the config servers has an impact on the rest of the system. 
