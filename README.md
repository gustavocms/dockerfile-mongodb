## A mongodb image based on phusion/baseimage

A simple mongodb image based on phusion/baseimage, capable of integrate a complete sharded mongodb cluster.

### Getting Started

Here are basic instructions to initialize your container in a variety of ways

* To simply run the container in deamon mode and expose ports

sudo docker run -d -P gustavocms/mongodb

* To run forcing the ports to be exposed with specific binds (in this case, the same number). It works with 1 container per host

sudo docker run -d -p 27017:27017 -p 28017:28017 gustavocms/mongodb

* To run specifying the name of the running container

sudo docker run -d -P gustavocms/mongodb --name rs1_srv1

* The parameters --noprealloc and --smallfiles are optional to avoid MongoDB is consuming lots of memory (useful while testing).

sudo docker run -d -P gustavocms/mongodb --noprealloc --smallfiles

* As part of a replicaset (for high availability)

sudo docker run -d -P gustavocms/mongodb --replSet rs1

### Docker and MongoDB Sharded Cluster

This article describes how to create a Mongo DB Sharded Cluster using Docker, as seen in http://sebastianvoss.com/docker-mongodb-sharded-cluster.html

* Create the Replica Sets
For our shard we need some replica sets. We will create two of them consisting of three nodes each. These commands will instantiate three mongod containers and assign them to replica set rs1. 

sudo docker run -P -name rs1_srv1 -d gustavocms/mongodb --replSet rs1 
sudo docker run -P -name rs1_srv2 -d gustavocms/mongodb --replSet rs1 
sudo docker run -P -name rs1_srv3 -d gustavocms/mongodb --replSet rs1 

sudo docker run -P -name rs2_srv1 -d gustavocms/mongodb --replSet rs2 
sudo docker run -P -name rs2_srv2 -d gustavocms/mongodb --replSet rs2 
sudo docker run -P -name rs2_srv3 -d gustavocms/mongodb --replSet rs2 

Create as many replica set shards as needed

* Inspect all the new containers
Take a note of the IP address and port bindings of all Docker containers.

sudo docker inspect rs1_srv1
sudo docker inspect rs1_srv2

* Initialize the Replica Sets
Connect to MongoDB running in container rs1_srv1 (you need the local port bound for 27017/tcp figured out in the previous step).

mongo --port <port>

$ MongoDB shell
rs.initiate()
rs.add("<IP_of_rs1_srv2>:27017")
rs.add("<IP_of_rs1_srv3>:27017")
rs.status()

Docker will assign an auto-generated hostname for containers and by default MongoDB will use this hostname while initializing the replica set. As we need to access the nodes of the replica set from outside the container we will use IP adresses instead.

Use these MongoDB shell commands to change the hostname to the IP address.

$ MongoDB shell
cfg = rs.conf()
cfg.members[0].host = "<IP_of_rs1_srv1>:27017"
rs.reconfig(cfg)
rs.status()

Repeat these process with all replica sets.

* Create some Config Servers
Now we create three MongoDB config servers to manage our shard. For development one config server would be sufficient but this shows how easy it is to start more.

sudo docker run -P -name cfg1 -d gustavocms/mongodb --configsvr --dbpath /data/db --port 27017
sudo docker run -P -name cfg2 -d gustavocms/mongodb --configsvr --dbpath /data/db --port 27017
sudo docker run -P -name cfg3 -d gustavocms/mongodb --configsvr --dbpath /data/db --port 27017

* Create a Router (from now on using gustavocms/mongos, a distinct image)
A MongoDB router is the point of contact for all the clients of the shard. The --configdb parameter takes a comma separated list of IP addresses and ports of the config servers.
Note that we are using the mongos image created in the first part.

sudo docker run -P -name mongos1 -d gustavocms/mongos --port 27017 --configdb 
    <IP_of_container_cfg1>:27017, \
    <IP_of_container_cfg2>:27017, \
    <IP_of_container_cfg3>:27017
    
Initialize the Shard
Finally we need to initialize the shard. Connect to the MongoDB router (you need the local port bound for 27017/tcp) and execute these shell commands:

mongo --port <port>

# MongoDB shell

sh.addShard("rs1/<IP_of_rs1_srv1>:27017")
sh.addShard("rs2/<IP_of_rs2_srv1>:27017")
sh.status()
