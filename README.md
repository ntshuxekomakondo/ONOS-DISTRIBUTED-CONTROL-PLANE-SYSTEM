# ONOS-DISTRIBUTED-CONTROL-PLANE-SYSTEM

This guide explains how to deploy a distributed Software-Defined Networking (SDN) control plane using the Open Network Operating System (ONOS) version 2.5.1 and Atomix version 3.1.9 for cluster coordination.
The setup is tested in Flat architectures using Mininet for network emulation.

ONOS 2.5.1 Distributed Control Plane with Atomix 3.1.9

Deployment Guide — 3 Atomix Nodes + 4 ONOS Nodes

1. Overview

This guide provides step-by-step instructions for deploying a homogeneous distributed SDN control plane using the Open Network Operating System (ONOS) version 2.5.1 with Atomix version 3.1.9 as the distributed coordination backend.

The setup uses:

3 Atomix nodes (Raft consensus cluster) for distributed state replication.

4 ONOS controller nodes connected to the Atomix cluster.

All nodes run Ubuntu 18.04.6 LTS with Java 11.

Atomix nodes must be started first to establish leader election and consensus before launching ONOS.

2. Architecture

Network: 192.168.1.0/24

Node	Role	Hostname	IP Address
1	Atomix-1	atomix-node1	192.168.1.1
2	Atomix-2	atomix-node2	192.168.1.2
3	Atomix-3	atomix-node3	192.168.1.3
4	ONOS-1	onos-node1	192.168.1.4
5	ONOS-2	onos-node2	192.168.1.5
6	ONOS-3	onos-node3	192.168.1.6
7	ONOS-4	onos-node4	192.168.1.7
3. Requirements
3.1 Operating System

All nodes run:

Distributor ID: Ubuntu  
Description:    Ubuntu 18.04.6 LTS  
Release:        18.04  
Codename:       bionic

3.2 Java

Install Java 11 on all nodes:

sudo apt update
sudo apt install openjdk-11-jdk -y
java -version


Expected output:

openjdk version "11.x.x"
OpenJDK Runtime Environment (build 11.x.x)
OpenJDK 64-Bit Server VM (build 11.x.x)

3.3 Hardware

Atomix Nodes (3 total):

2 vCPUs

4 GB RAM

20 GB SSD

1 Gbps NIC

ONOS Nodes (4 total):

4 vCPUs

8 GB RAM

20 GB SSD

1 Gbps NIC

4. Installation
4.1 Install Atomix 3.1.9 (on Atomix nodes only)
sudo mkdir /opt/atomix && cd /opt/atomix
sudo wget https://repo1.maven.org/maven2/io/atomix/atomix-dist/3.1.9/atomix-dist-3.1.9.tar.gz
sudo tar -xzf atomix-dist-3.1.9.tar.gz
sudo mv atomix-dist-3.1.9 atomix

4.2 Configure Atomix Cluster

On each Atomix node, edit /opt/atomix/conf/atomix.conf and update node.id and address accordingly:

Example for atomix-node1:

cluster {
  cluster-id: "onos"

  node {
    id: "atomix-1"
    address: "192.168.1.1:5679"
  }

  discovery {
    type: "bootstrap"
    nodes.1 {
      id: "atomix-1"
      address: "192.168.1.1:5679"
    }
    nodes.2 {
      id: "atomix-2"
      address: "192.168.1.2:5679"
    }
    nodes.3 {
      id: "atomix-3"
      address: "192.168.1.3:5679"
    }
  }
}

management-group {
  type: "raft"
  partitions: 1
  storage.level: "disk"
  members: ["atomix-1", "atomix-2", "atomix-3"]
}

partition-groups.raft {
  type: "raft"
  partitions: 3
  storage.level: "disk"
  members: ["atomix-1", "atomix-2", "atomix-3"]
}


Repeat for atomix-node2 and atomix-node3, changing node.id and address to match the respective IP.

4.3 Start Atomix Nodes
cd /opt/atomix/bin
./atomix-agent ../conf/atomix.conf


Check logs for leader election:

INFO  RaftServer: Leader elected: atomix-2


Other nodes will have Follower or Candidate roles depending on Raft state.

4.4 Install ONOS 2.5.1 (on ONOS nodes only)
sudo mkdir /opt/onos && cd /opt/onos
sudo wget https://repo1.maven.org/maven2/org/onosproject/onos-releases/2.5.1/onos-2.5.1.tar.gz
sudo tar -xzf onos-2.5.1.tar.gz
sudo mv onos-2.5.1 onos

4.5 Configure ONOS Cluster

On each ONOS node, create /opt/onos/config/cluster.json:

Example for onos-node1:

{
  "name": "onos",
  "node": {
    "id": "onos-1",
    "ip": "192.168.1.4",
    "port": 9876
  },
  "storage": [
    { "id": "atomix-1", "ip": "192.168.1.1", "port": 5679 },
    { "id": "atomix-2", "ip": "192.168.1.2", "port": 5679 },
    { "id": "atomix-3", "ip": "192.168.1.3", "port": 5679 }
  ]
}


Repeat for ONOS nodes 2–4, changing id and ip.

4.6 Start ONOS Controllers
/opt/onos/bin/onos-service start


Verify cluster formation:

onos-cli
nodes


Expected:

id=onos-1, address=192.168.1.4, state=READY
id=onos-2, address=192.168.1.5, state=READY
id=onos-3, address=192.168.1.6, state=READY
id=onos-4, address=192.168.1.7, state=READY

5. Notes on Raft Roles

Leader: Handles all write requests, coordinates log replication.

Follower: Passive, receives log entries from Leader.

Candidate: Temporary state during leader election.

Atomix must have at least 3 nodes to maintain quorum in Raft consensus (majority = 2). This ensures fault tolerance and prevents split-brain.

6. Recommendations

Always start Atomix nodes first and confirm leader election before launching ONOS.

Use dedicated NICs for control plane traffic in production.

Monitor cluster health with onos-top and Atomix logs.
