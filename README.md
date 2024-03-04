# Cassandra Multi-Node

## Prerequisites:
2 Ubuntu Servers(22.04)

```
Node1: 192.168.56.146
Node2: 192.168.56.148
```

## Java Installation: You should install java on both nodes

**On node1 and node2**
```
sudo apt update
sudo apt install default-jre
sudo apt install default-jdk
```

**Verify installation**:

```
java -version
openjdk version "11.0.22" 2024-01-16
OpenJDK Runtime Environment (build 11.0.22+7-post-Ubuntu-0ubuntu222.04.1)
OpenJDK 64-Bit Server VM (build 11.0.22+7-post-Ubuntu-0ubuntu222.04.1, mixed mode, sharing)

```


## Cassandra Installation: You should install Cassandra on both nodes 

**On node1 and node2**
```
sudo apt update
sudo apt upgrade
```


**Add Cassandra Repository**
```
echo "deb https://debian.cassandra.apache.org 40x main" | sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
wget -q -O - https://www.apache.org/dist/cassandra/KEYS | sudo tee /etc/apt/trusted.gpg.d/cassandra.asc
apt update
```

**Install Cassandra**
```
apt install cassandra
systemctl status cassandra
nodetool status
root@node1:~# cqlsh
root@node2:~# cqlsh
```


## Configuring the Firewall to Allow Cassandra Traffic

ufw enable

**On node1**
```
sudo ufw allow from 192.168.56.148 to 192.168.56.146 proto tcp port 7000,9042
```

**On node2**
```
sudo ufw allow from 192.168.56.146 to 192.168.56.148 proto tcp port 7000,9042
```
**Check ufw open Ports**

```
ufw status numbered
```


## Cassandra Multi-Node Installation

**On node1 and node2**

```
root@node1:~# systemctl stop cassandra
root@node1:~# rm -rf /var/lib/cassandra/*

root@node2:~# systemctl stop cassandra
root@node2:~# rm -rf /var/lib/cassandra/*

```

**On Node1:**
```
root@node1:~# vim  /etc/cassandra/cassandra.yaml
...
cluster_name: 'MyCassandra'
...
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
         - seeds: "192.168.56.146"
...
listen_address: "192.168.56.146"
...
rpc_address: "192.168.56.146"
...
endpoint_snitch: GossipingPropertyFileSnitch
...
auto_bootstrap: false

root@node1:~# systemctl start cassandra
```

**On Node2:**

```
vim  /etc/cassandra/cassandra.yaml
...
cluster_name: 'MyCassandra'
...
seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
         - seeds: "192.168.56.146"
...
listen_address: "192.168.56.148"
...
rpc_address: "192.168.56.148"
...
endpoint_snitch: GossipingPropertyFileSnitch
...
auto_bootstrap: false


root@node2:~# systemctl start cassandra
```


**Connecting to Multi-Node Cassandra Cluster**

**On Node1:**
```
root@node1:~# nodetool status

Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address         Load       Tokens  Owns (effective)  Host ID                               Rack
UN  192.168.56.146  69.03 KiB  16      100.0%            909eac6e-58f5-46df-aa8a-c211b8e7d8ee  rack1
UN  192.168.56.148  69.02 KiB  16      100.0%            c5c1f27f-44ad-4590-8c9f-818579072f6d  rack1
```

**On Node2:**
```
root@node2:~# nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving
--  Address         Load        Tokens  Owns (effective)  Host ID                               Rack
UN  192.168.56.146  200.9 KiB   16      48.8%             909eac6e-58f5-46df-aa8a-c211b8e7d8ee  rack1
UN  192.168.56.148  196.57 KiB  16      51.2%             c5c1f27f-44ad-4590-8c9f-818579072f6d  rack1
```


## Testing Database:

**On Node1:**
```
root@node1:~# cqlsh 192.168.56.146 9042

Connected to MyCassandra at 192.168.56.146:9042
[cqlsh 6.0.0 | Cassandra 4.0.12 | CQL spec 3.4.5 | Native protocol v5]
Use HELP for help.
cqlsh> describe cluster
Cluster: MyCassandra
Partitioner: Murmur3Partitioner
Snitch: DynamicEndpointSnitch
```
**On Node2:**
```
root@node1:~# cqlsh 192.168.56.148 9042
Connected to MyCassandra at 192.168.56.148:9042
[cqlsh 6.0.0 | Cassandra 4.0.12 | CQL spec 3.4.5 | Native protocol v5]
Use HELP for help.
cqlsh> describe cluster

Cluster: MyCassandra
Partitioner: Murmur3Partitioner
Snitch: DynamicEndpointSnitch
```


## Create a table and insert data


```
root@node1:~# cqlsh 192.168.56.146 9042
cqlsh> CREATE KEYSPACE IF NOT EXISTS Company WITH REPLICATION = {'class':'SimpleStrategy', 'replication_factor' : 1 };
cqlsh> USE Company;
cqlsh:company> CREATE TABLE IF NOT EXISTS employee (emp_id   ascii PRIMARY KEY,emp_name   varchar,emp_city varchar,emp_phone text,emp_sal text,);
cqlsh:company> INSERT INTO employee(emp_id, emp_name,emp_city,emp_phone,emp_sal) VALUES('8080','Soroush','Rasht','093700000','60000000') IF NOT EXISTS;

cqlsh:company> SELECT * FROM employee;

 emp_id | emp_city | emp_name | emp_phone   | emp_sal
--------+----------+----------+-------------+----------
   8080 |    Rasht |  Soroush | 093700000 | 60000000
```


## Check Database syncing

**On Node1:**
```
root@node1:~# cqlsh 192.168.56.146 9042
Connected to MyCassandra at 192.168.56.146:9042
[cqlsh 6.0.0 | Cassandra 4.0.12 | CQL spec 3.4.5 | Native protocol v5]
Use HELP for help.
cqlsh> use company;
cqlsh:company> SELECT * FROM employee;

 emp_id | emp_city | emp_name | emp_phone   | emp_sal
--------+----------+----------+-------------+----------
   8080 |    Rasht |  Soroush | 093700000 | 60000000

(1 rows)

```

**On Node2:**
```
root@node2:~# cqlsh 192.168.56.148 9042
Connected to MyCassandra at 192.168.56.148:9042
[cqlsh 6.0.0 | Cassandra 4.0.12 | CQL spec 3.4.5 | Native protocol v5]
Use HELP for help.
cqlsh> USE Company;
cqlsh:company> SELECT * FROM employee;

 emp_id | emp_city | emp_name | emp_phone   | emp_sal
--------+----------+----------+-------------+----------
   8080 |    Rasht |  Soroush | 093700000 | 60000000

(1 rows)
```


