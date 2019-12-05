# mongosharding

# localhost sharding in mongo 
```sh
$ mkdir rs-a-1
$ mkdir rs-a-2
$ mkdir rs-a-3
$ mkdir rs-b-1
$ mkdir rs-b-2
$ mkdir rs-b-3
$ mkdir old
$ mkdir old2
$ mkdir oldarb
```
# old cluster
```sh
$ mongod  --replSet old --dbpath old --port 27017 --logpath old.log --fork
$ mongod  --replSet old --dbpath old2 --port 27018 --logpath old2.log --fork
$ mongod  --replSet old --dbpath oldarb --port 27024 --logpath oldarb.log --fork

mongo localhost:27017
rs.initiate()
rs.add("localhost:27018")
rs.addArb("localhost:27024")

sharding a/b
$ mongod --shardsvr --replSet shard-a --dbpath rs-a-1 --port 30000 --logpath rs-a-1.log --fork
$ mongod --shardsvr --replSet shard-a --dbpath rs-a-2 --port 30001 --logpath rs-a-2.log --fork
$ mongod --shardsvr --replSet shard-a --dbpath rs-a-3 --port 30002 --logpath rs-a-3.log --fork

mongo localhost:30000
rs.initiate()
rs.add("localhost:30001")
rs.addArb("localhost:30002")

$ mongod --shardsvr --replSet shard-b --dbpath rs-b-1 --port 30100 --logpath rs-b-1.log --fork
$ mongod --shardsvr --replSet shard-b --dbpath rs-b-2 --port 30101 --logpath rs-b-2.log --fork
$ mongod --shardsvr --replSet shard-b --dbpath rs-b-3 --port 30102 --logpath rs-b-3.log --fork

mongo localhost:30100
rs.initiate()
rs.add("localhost:30101")
rs.addArb("localhost:30102")

config server 
$ mkdir config-1
$ mongod --configsvr --dbpath config-1 --replSet config_replica --port 27019 --logpath config-1.log --fork
$ mkdir config-2
$ mongod --configsvr --dbpath config-2 --replSet config_replica --port 27020 --logpath config-2.log --fork
$ mkdir config-3
$ mongod --configsvr --dbpath config-3 --replSet config_replica --port 27021 --logpath config-3.log --fork

mongo localhost:27019
rs.initiate()
rs.add("localhost:27020")
rs.add("localhost:27021")

mongos server
mongos --configdb "config_replica/localhost:27019,localhost:27020,localhost:27021"  --logpath mongos.log --fork --port 40000

connection to mongos server
mongo localhost:40000
> sh.addShard("shard-a/localhost:30000,localhost:30001")
> sh.addShard("shard-b/localhost:30100,localhost:30101")
```

> db.getSiblingDB("config").shards.find()
> use admin
> db.runCommand({listshards: 1})


# Enable Sharding  & monitoring

> sh.enableSharding("cloud-docs")
> db.getSiblingDB("config").databases.find()

select the shard key by most of all hit ratio on collection
> sh.shardCollection("cloud-docs.spreadsheets", {username: 1, _id: 1})

```sh
------------Studio 3T IDE--------------------------------------
use cloud-docs
sh.enableSharding("cloud-docs")
sh.shardCollection("cloud-docs.spreadsheets",{_id:1,username:1},true)


for(i=1;i<=300;i++){
    db.spreadsheets.insert({_id:i,username:"abdul"+i})
}

for(i=301;i<=400;i++){
    db.spreadsheets.insert({_id:i,username:"Buetter"+i})
}

for(i=401;i<=500;i++){
    db.spreadsheets.insert({_id:i,username:"hawkins"+i})
}


for(i=501;i<=600;i++){
    db.spreadsheets.insert({_id:i,username:"Buetter"+i})
}

for(i=601;i<=700;i++){
    db.spreadsheets.insert({_id:i,username:"hawkins"+i})
}


db.spreadsheets.deleteMany({})

db.spreadsheets.find({_id:1}).explain()
db.spreadsheets.stats()
sh.status()

use config
db.chunks.count({"shard": "shard-a"})
db.chunks.count({"shard": "shard-b"})

```

## zero downtime recovery

# from old db
```sh
mongodump  --out dump --host localhost --port 27017 --db cloud-docs
```
# oplog db ozelinde çalışmaz full db'yi indirir
```sh
mongodump  --out dump --oplog  --host localhost --port 27017 
```

# oplog replay

```sh
$ mkdir -p oplog_tmp
$ rm -rf oplog_tmp/*
$ cp dump/oplog.bson oplog_tmp/
$ mongorestore --host localhost --port 40000 oplog_tmp
```

# to new sharded db
```sh
$ mongorestore  --host localhost --port 40000 dump
```

# failover
rs.stepDown(20) -> 20sn kadar primary olan makina secondary olur ve bakımı yapılır.Vote efektif olması için 3 node ile çalışmak gerekir.Bu durumda tek replica oldugu için vote efektif çalışmaz ise rs.freeze() komutu ile arbiter'ın diger makinayı primary olarak seçmesi enfors edilebilir.
rs.freeze() -> refresh and arbiter'ın diger makinayı primary yapması için.
