# EC4Docker (Elastic Cluster 4 Docker)

__EC4Docker__ is a simple Torque based Elastic Cluster whose nodes are contaniers. There exists a front-end that can be accessed by ssh, and the internal _working nodes_ are powered on or off according to the needs (if the nodes are not used for a while, they are powered off, and they are powered on if they are needed).

Features of the cluster:
- Front end that has SSH access.
- Passwordless SSH access from frontend to the working nodes.
- Customizable number of working nodes.
- Self-managed elasticity by using [CLUES](https://github.com/grycap/clues).
- Shared filesystem from frontend to working nodes by using NFS

## How to use it
1. Create your front-end and working node docker images. 
2. Edit the _ec4docker.config_ file to configure the cluster.
3. Use _setup-cluster_ script to start the cluster.
4. Enter into the cluster.
 
## Building the docker images
You can build the _front-end_ and _working node_ docker images by issuing the following commands:

```bash
docker build -t ec4docker:frontend -f frontend/Dockerfile.clues frontend/
docker build -t ec4docker:wn wn
```

The images will be built and registered in your local registry. Their names are _ec4dockerclues:frontend_ and _ec4docker:wn_.

Alternatively you can build the non-elastic version: _ec4docker:frontend_ that does not install CLUES by issuing the following commands:

```bash
docker build -t ec4docker:frontend -f frontend/Dockerfile frontend/
docker build -t ec4docker:wn wn
```

In this case you need to power the nodes on or of by hand (using the provided scripts in folder _/opt/ec4docker_).

__NOTE__: you are advised to modify the Dockerfile files in order to include your libraries, applications, etc. to customize your cluster.

## Configure the cluster
You should edit the file _ec4docker.config_ file to set the name of your cluster (this name will be set for the front-end node in docker), the base name for the working nodes (they should be named as _basename_1, _basename_2, etc.) and the max amount of computing nodes. You must also set the names of the docker images according to the previous step.

A simple example for this file is:
```bash
EC4DOCK_SERVERNAME=ec4docker
EC4DOCK_MAXNODES=4
EC4DOCK_FRONTEND_IMAGENAME=ec4docker:frontend
EC4DOCK_WN_IMAGENAME=ec4docker:wn
EC4DOCK_NODEBASENAME=ec4dockernode
```

In this file the cluster will be named _ec4docker_.

## Create the cluster
You can use the script _setup-cluster_ to create the front-end of the cluster, from the corresponding docker image. If the cluster already exists, this script will ask you to kill it.

__IMPORTANT__: In order to be able to use the NFS shared filesystem, you __MUST__ enable nfsd module in the kernel of the docker servers that hosts the containers.
```bash
$ modprobe nfsd
```

__NOTE__: The settings of the clusters are those that are set in file _ec4docker.config_ file. Take note of those settings because you will need them in order to access the cluster. In special, the name of the cluster which is in _EC4DOCK_SERVERNAME_.

__WARNING__: The cluster is created on a _Docker aside Docker_ approach. That means that the front-end will issue docker calls to create and to destroy the docker containers that will serve as working nodes from the cluster. But these docker containers will be created in the docker host that started the front-end. In order to use this approach, the docker communication socket and the docker binary from the host are shared with the container.

## Enter the cluster
Once the front-end has been created you can enter into the front-end container by issuing a command like the next one (the name of the container depends on your configuration; i.e. the _ec4docker.config_ file):

```bash
$ docker exec -it torqueserver /bin/bash
```

Then you can _su_ as the __ubuntu__ user (which is the only user created in the cluster) and issue commands to the queue. CLUES will intercept the call and will power on some working nodes in the cluster.

An example is the next:
```bash
$ su - ubuntu
$ echo "hostname && sleep 10" | qsub
1.ec4docker
$ qstat                             
Job id                    Name             User            Time Use S Queue
------------------------- ---------------- --------------- -------- - -----
1.ec4docker               STDIN            ubuntu                 0 R batch 
$ ls -l
total 4
-rw------- 1 ubuntu ubuntu  0 Feb 12 11:15 STDIN.e1
-rw------- 1 ubuntu ubuntu 13 Feb 12 11:15 STDIN.o1
```

__NOTE__: For the non-elasic version, you can power on some nodes from inside the front-end, by hand by issuing commands like the next:
```bash
$ /opt/ec4docker/poweron ec4dockernode1
$ /opt/ec4docker/poweron ec4dockernode2
```
