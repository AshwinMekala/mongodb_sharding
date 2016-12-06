# mongodb_sharding
MongoDB Sharding &amp; Benchmarking


-------------------------------------
Environment
-------------------------------------

MongoDB 3.2
Linux CentOS 7 (Virtual Box)
10GB SSD
1GB RAM
1 Core Processor

+++++++++++++++++++++++++++++++++++++
Set Up
+++++++++++++++++++++++++++++++++++++

-------------------------------------
config1
-------------------------------------

mongod -config /etc/mongod-config1.conf
mongo --host 127.0.0.1 --port 27020

rs.initiate(
  {
    _id: "config1",
    configsvr: true,
    members: [
      { _id : 0, host : "127.0.0.1:27020" }
    ]
  }
)

-------------------------------------
shard1
-------------------------------------

mongod -config /etc/mongod-shard1.conf
mongo --host 127.0.0.1 --port 27018

rs.initiate(
  {
    _id : "shard1",
    members: [
      { _id : 0, host : "127.0.0.1:27018" }
    ]
  }
)

-------------------------------------
shard2
-------------------------------------

mongod -config /etc/mongod-shard2.conf
mongo --host 127.0.0.1 --port 27019

rs.initiate(
  {
    _id : "shard2",
    members: [
      { _id : 0, host : "127.0.0.1:27019" }
    ]
  }
)


------------------------------------------------
Creating mongos router:
------------------------------------------------

mongos -config /etc/mongod-mongos1.config

------------------------------------------------
adding shards
------------------------------------------------

mongo 127.0.0.1:27021

sh.addShard( "shard1/127.0.0.1:27018")
sh.addShard( "shard2/127.0.0.1:27019")

------------------------------------------------
insert data into collection
------------------------------------------------

for(var i=0; i<500000; i++){
  db.restaurants_hash.save({restaurant_id:i, about_restaurant:"0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
}

------------------------------------------------
sharding collection
------------------------------------------------
sh.enableSharding("restaurants")
db.restaurants.ensureIndex( { restaurant_id : "hashed" } )
sh.shardCollection("restaurants.restaurants", { restaurant_id : "hashed" } )


------------------------------------------------

db.restaurants.getShardDistribution()

Shard shard1 at shard1/127.0.0.1:27018
 data : 48.1MiB docs : 300272 chunks : 2
 estimated data per chunk : 24.05MiB
 estimated docs per chunk : 150136

Shard shard2 at shard2/127.0.0.1:27019
 data : 31.99MiB docs : 199728 chunks : 1
 estimated data per chunk : 31.99MiB
 estimated docs per chunk : 199728

Totals
 data : 80.1MiB docs : 500000 chunks : 3
 Shard shard1 contains 60.05% data, 60.05% docs in cluster, avg obj size on shard : 168B
 Shard shard2 contains 39.94% data, 39.94% docs in cluster, avg obj size on shard : 168B

? we get lower uniform distribution if chunk size is high. But negligible if data is large.

Drop database:
db.restaurants_hash.drop()

------------------------------------------------
Decrease Chunk size
------------------------------------------------

use config
db.settings.save( { _id:"chunksize", value: 1 } )

? we get higher uniform distribution with lower chunk size. But higher chunk migrations.
  for demonstration we use minimum chunk size of 1MB. 

------------------------------------------------
Hashed Vs Range Sharding Inserting
------------------------------------------------

for(var i=0; i<100000; i++){
  db.restaurants.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
}

for(var i=0; i<100000; i++){
  db.restaurants_hash.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
}

for(var i=0; i<100000; i++){
	db.restaurants_range.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
}

------------------------------------------------
Sharding as hash & range
------------------------------------------------

db.restaurants.createIndex( { restaurant_id : "hashed" })

// hash sharding
db.restaurants_hash.createIndex( { restaurant_id : "hashed" })
sh.shardCollection("restaurants.restaurants_hash", { restaurant_id : "hashed" })
db.restaurants_hash.getShardDistribution()

// range sharding
db.restaurants_range.createIndex( { restaurant_id : 1 })
sh.shardCollection("restaurants.restaurants_range", { restaurant_id : 1 })
db.restaurants_range.getShardDistribution()

? no. of chunks higher in range sharding than hash sharding. So higher migration in range sharding.

sh.status()

? proof for higher migration in range sharding. 62 out of 96 chunks migrated are range based sharding.

db.restaurants.status()
db.restaurants_hash.status()
db.restaurants_range.status()

? compare index sizes in bytes

------------------------------------------------
Find/Select Indexed Query Time Analysis
------------------------------------------------

Test 1: 40K documents

function countTime(){
  var start = new Date().getTime();
  db.restaurants.find({restaurant_id : { $gte: 0, $lt: 40000 }}).count();
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_hash.find({restaurant_id : { $gte: 0, $lt: 40000 }}).count();
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_range.find({restaurant_id : { $gte: 0, $lt: 40000 }}).count();
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

Non Hash  Range
20   62   18
19   65   20
22   56   18
18   70   21
28   51   22

Test 2: 100K documents

function countTime(){
  var start = new Date().getTime();
  db.restaurants.find({restaurant_id : { $gte: 0, $lt: 100000 }}).count();
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_hash.find({restaurant_id : { $gte: 0, $lt: 100000 }}).count();
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_range.find({restaurant_id : { $gte: 0, $lt: 100000 }}).count();
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

Non Hash  Range
20   80   25
25   75   27
31   65   23
24   57   30
21   62   35

------------------------------------------------
Find/Select Query Time Analysis
------------------------------------------------

Test 1: 40K documents

function countTime(){
  var start = new Date().getTime();
  db.restaurants.find({restaurant_id : { $gte: 0, $lt: 40000 }}).toArray();
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_hash.find({restaurant_id : { $gte: 0, $lt: 40000 }}).toArray();
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_range.find({restaurant_id : { $gte: 0, $lt: 40000 }}).toArray();
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

Non Hash  Range
350  429  397
445  467  330
374  437  389
434  428  442
465  454  433

Test 2: 100K documents

function countTime(){
  var start = new Date().getTime();
  db.restaurants.find({restaurant_id : { $gte: 0, $lt: 100000 }}).toArray();
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_hash.find({restaurant_id : { $gte: 0, $lt: 100000 }}).toArray();
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  db.restaurants_range.find({restaurant_id : { $gte: 0, $lt: 100000 }}).toArray();
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();


Non   Hash   Range
1026  1061   1052
1008  1105   1033
984   1116   1021
1010  1027   1039
1031  1102   1084

------------------------------------------------
Insert/Create Query Time Analysis
------------------------------------------------

Test 1: 50K documents

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
  }
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants_hash.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
  }
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants_range.save({restaurant_id:i, about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz"});
  }
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();


Non     Hash     Range
21607  22486     20423
22322  23965     20671
21982  23358     19836
22241  24872     21628
20108  24601     19862

------------------------------------------------
Remove/Delete Query Time Analysis
------------------------------------------------

Test 1: 50K documents

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants.remove({restaurant_id:i});
  }
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants_hash.remove({restaurant_id:i});
  }
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=100000; i<150000; i++){
    db.restaurants_range.remove({restaurant_id:i});
  }
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

Non     Hash     Range
21663  27437     24305
20872  26795     24101
22449  29491     25746
21314  30101     26682
20808  28106     25286


------------------------------------------------
Update Query Time Analysis
------------------------------------------------

Test 1: 40K documents

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<40000; i++){
    db.restaurants.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<40000; i++){
    db.restaurants_hash.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<40000; i++){
    db.restaurants_range.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();


Non     Hash     Range
15939   17583    16606
15995   16707    16497
15962   17046    16778
16012   17201    17010
15887   16902    16304


Test 2: 100K documents

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<100000; i++){
    db.restaurants.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Non Shard Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<100000; i++){
    db.restaurants_hash.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Hash Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();

function countTime(){
  var start = new Date().getTime();
  for(var i=0; i<100000; i++){
    db.restaurants_range.update({restaurant_id:i}, { $set: { about_restaurant: "0123456789ABCDEFGHIJKLMNOPQRSTUVWXTZabcdefghiklmnopqrstuvwxyz" } });
  }
  print("Range Time taken msecs: "+ (new Date().getTime() - start) );
}countTime();


Non     Hash     Range
43599   45709    42742
42787   46105    42301
42304   47524    43001
41872   46324    42247
42545   56871    43146


------------------------------------------------
Stop mongo processes
------------------------------------------------

ps -edaf | grep mongo | grep -v grep
kill <PROCESS_ID>

------------------------------------------------
Open Source Project Files
------------------------------------------------

https://github.com/ashwinmekala/mongodb_sharding
