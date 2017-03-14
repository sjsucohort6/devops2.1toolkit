# DevOps 2.1 Toolkit Notes

Notes from DevOps 2.1 Toolkit book by Victor Farcic

# Chapter 1 - CI using Docker

**git clone https://github.com/vfarcic/go-demo.git
cd go-demo**

## create a VM with virtualbox driver and named go-demo
**docker-machine create -d virtualbox go-demo**

**docker-machine env go-demo**

**eval $(docker-machine env go-demo)**


cat docker-compose-test-local.yml

```
version: '2'

services:

  app:
    build: .
    image: go-demo

  unit:
    image: golang:1.6
    volumes:
      - .:/usr/src/myapp
      - /tmp/go:/go
    working_dir: /usr/src/myapp
    command: bash -c "go get -d -v -t && go test --cover -v ./... && go build -v -o go-demo"

  staging-dep:
    image: go-demo
    ports:
      - 8080:8080
    depends_on:
      - db

  staging:
    extends:
      service: unit
    environment:
      - HOST_IP=staging-dep:8080
    command: bash -c "go get -d -v -t && go test --tags integration -v"

  production:
    extends:
      service: unit
    environment:
      - HOST_IP=${HOST_IP}
    network_mode: host
    command: bash -c "go get -d -v -t && go test --tags integration -v"

  db:
    image: mongo:3.2.10
    
```

## Run unit test

**docker-compose -f docker-compose-test-local.yml run --rm unit**

## Build app image
**docker-compose -f docker-compose-test-local.yml build app**

## Run staging test
### run the dependencies
**docker-compose -f docker-compose-test-local.yml up -d staging-dep**

**docker-compose -f docker-compose-test-local.yml ps**
==> will show mongodb and go-demo app running

### run the service
**docker-compose -f docker-compose-test-local.yml run --rm staging**

## Remove the containers
Once tests completed we can remove the containers.

**docker-compose -f docker-compose-test-local.yml down**

## Pushing image to registry
We use Docker Registry hosted locally to store our images

```
registry:
    container_name: registry
    image: registry:2.5.0
    ports:
      - 5000:5000
    volumes:
      - .:/var/lib/registry
    restart: always

```

### start the registry
docker-compose -f docker-compose-local.yml up -d registry

###Test the registry by pushing an image to it:
docker pull alpine

docker tag alpine localhost:5000/alpine

docker push localhost:5000/alpine

ls -l docker/registry/v2/repositories/alpine

docker tag go-demo localhost:5000/go-demo:1.0

docker push localhost:5000/go-demo:1.0

### Push the app image to registry

docker tag go-demo localhost:5000/go-demo:1.0

docker push localhost:5000/go-demo:1.0

At this point we have completed the Continuous integration (CI) where the build has been unit and stage tested and is ready for integration testing manually.

But if we want to go the extra mile and implement continuous delivery (CD) then we need to do away with the manual aspect and automate integration test and deployment to production.

# Chapter 2 - Setting up Swarm Cluster
##Axis Scaling

1. X-Axis scaling - horizontal replicas or duplicates
2. Y-Axis scaling - vertical functional decomposition (or microservices)
3. Z-Axis scaling - Data partitioning - or sharding (mostly applicable to databases)

Server cluster = acts as one big server; applications deployed on a cluster could be on any node in the cluster

## Docker Swarm Mode
1. As of 1.12 docker release we have swarm mode where everything needed to manage the cluster of nodes is incorporated on docker engine itself and no swarm agent/container is required to be run on each node in the cluster (and be hooked up with a separate service registry - like Consul or Zookeeper). 
2. Swarm has built-in service discovery (Consul) so no need to setup a separate service registy.
3. Also Swarm maintains the desired state of the number of replicas to be run for a given app. 
4. Bundle used to define services are created from Docker compose files and it is same in both development and production.

See [docker-swarm.sh] (https://gist.github.com/vfarcic/750fc4117bad9d8619004081af171896)

### Create 3 nodes
```
for i in 1 2 3; do
    docker-machine create -d virtualbox node-$i
done
```

docker-machine ls => will list all 3 nodes.

### Init node-1 as manager
```
eval $(docker-machine env node-1)

docker swarm init 
    --advertise-addr $(docker-machine ip node-1)
```
### Command to get join-token 
#### Manager token
docker swarm join-token -q manager

#### Worker token
```
docker swarm join-token -q worker

export TOKEN=$(docker swarm join-token -q worker)
```

#### Node 2 and 3 join cluster as workers
```
for i in 2 3; do
  eval $(docker-machine env node-$i)

  docker swarm join \
    --token $TOKEN \
    --advertise-addr $(docker-machine ip node-$i) \
    $(docker-machine ip node-1):2377
done
```

#### Check the cluster
eval $(docker-machine env node-1)

```
docker node ls
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
hjisdpq2ucd3s0ad0czqdjaa0    node-3    Ready   Active
stv9r3y3j5v1e1iqhw8mxr8n0    node-2    Ready   Active
zieikebz88oceranj1k7p5wjw *  node-1    Ready   Active        Leader
```

#### Create overlay network for cluster
```
docker network create --driver overlay go-demo

docker network ls
```
#### Deploy the service on cluster
```
docker service create --name go-demo-db  --network go-demo  mongo:3.2.10

docker service ls

docker service inspect go-demo-db

docker service create --name go-demo \
  -e DB=go-demo-db \
  --network go-demo \
  vfarcic/go-demo:1.0
  
docker service ls
```

#### Scale the service
docker service scale go-demo=5

docker service ls

#### Shutdown one node
docker-machine rm -f node-3

docker service ps go-demo ==> Will show the containers running on node-3 are now distributed among remaining 2 nodes.

#### Shutdown remaining nodes
docker-machine rm -f node-1 node-2

# Chapter 3 - Swram Networking And Reverse Proxy

See [networking.sh] (https://gist.github.com/vfarcic/fc88da0d389d3a8b56d666ea3f25cdc0)

Setup the 3 node cluster (as done above)

Requirements:
1. Load balancer (itself be fault tolerant) will distribute load across reverse proxies.
2. Reverse proxies will route the requests based on their base url to respective apps.
3. go-demo app will be accessible only through reverse proxy. It can talk to go-demo-db and the data of the DB will only be accessible through the APIs exposed by go-demo app.
4. go-demo-db will not be accessible to any other app but go-demo app alone.

## Create DB attached to network
Dont expose the DB port

```
docker service create --name go-demo-db \
  --network go-demo \
  mongo:3.2.10
```

## Create util test app attached to network 

This app is to enable us to exec on the container and see the networking worked.

```
docker service create --name util \
    --network go-demo --mode global \
    alpine sleep 1000000000
```

Global mode is to run the service named "util" on all nodes in the cluster.

**Get util container's ID on node-1:**

ID=$(docker ps -q --filter label=com.docker.swarm.service.name=util)

**Add drill utility:**

docker exec -it $ID apk add --update drill

Exec drill on one of the util containers to access go-demo-db. Since they both are connected via go-demo overlay network they should be able to talk.

docker exec -it $ID drill go-demo-db

*NOERROR and Answer: 1 means we are able to access go-demo-db.*

10.0.0.2 is IP assigned to go-demo-db service.

**Each service gets a virtual IP when attached to overlay network. So all containers of a given service get the same virtual IP.**

##Service through reverse proxy

###Create a proxy network

docker network create --driver overlay proxy

docker network ls -f "driver=overlay"

==> will show all overlay networks (go-demo, proxy and ingress, which is created by default)

### Create app service 

Join it to both the go-demo network (so it can talk to go-demo-db) and to the proxy network (so it can be accessible by proxy service.

```
docker service create --name go-demo \
  -e DB=go-demo-db \
  --network go-demo \
  --network proxy \
  vfarcic/go-demo:1.0
```

### Create Reverse Proxy 
It will route requests based on their base url.

```
docker service create --name proxy \
    -p 80:80 \
    -p 443:443 \
    -p 8080:8080 \
    --network proxy \
    -e MODE=swarm \
    vfarcic/docker-flow-proxy
```

[vfarcic/docker-flow-proxy](https://github.com/vfarcic/docker-flow-proxy)
is based on [HAProxy](https://store.docker.com/images/85c386ff-85a7-4d61-b309-5901f625c36f?tab=description) and adds feature to dynamically configure configuration using REST APIs when new containers are added or old ones are removed.

Docker networking makes sure that all requests are first sent to the proxy service.

MODE=swarm tells proxy that containers are deployed to swarm cluster.

### Reconfigure proxy for go-demo service request routing

curl "$(docker-machine ip node-1):8080/v1/docker-flow-proxy/reconfigure?serviceName=go-demo&servicePath=/demo&port=8080"

### Send request
curl -i $(docker-machine ip node-1)/demo/hello

curl -i $(docker-machine ip node-3)/demo/hello

In either of the above cases the request will be re-directed to proxy container (which could be running on any one of the nodes in the cluster) by docker networking's routing mesh and proxy will re-route the request to go-demo service app based on /demo URL.

### Get the proxy container ID
#### Get the node name
NODE=$(docker service ps proxy | tail -n +2 | awk '{print $4}')

#### Get container ID from node

eval $(docker-machine env $NODE)
ID=$(docker ps -q --filter "ancestor=vfarcic/docker-flow-proxy")

or 

ID=$(docker ps -q --filter label=com.docker.swarm.service.name=proxy)

#### Check HAProxy configuration
```
docker exec -it \
    $ID cat /cfg/haproxy.cfg
```

## Load Balancing Requests
across all instances of a service.

Just scale the service and relax. 

Docker swarm networking does the load balancing of requests to go-demo service instances (all 5 of them in round-robin fashion). 

Proxy does not have to load balance. It just needs to route the incoming request based on the base URL (/go-demo/*) to a service (go-demo).

So there is no need to update the proxy configuration every time there is a change in cluster configuration (node added, removed, failed etc).

Only when we add a new service that we need to update the proxy configuration to route requests for the new service.


```
eval $(docker-machine env node-1)

docker service scale go-demo=5

```

ID=$(docker ps -q --filter label=com.docker.swarm.service.name=util)

docker exec -it $ID apk add --update drill

# Chapter 4 - Service Discovery in Swarm Cluster

Docker engine (as of 1.12) acts as a:
1. Swarm Manager
2. Swarm Registry
3. Swarm Worker

as all of the above functionalities are bundled into the docker engine.

```
git clone https://github.com/vfarcic/cloud-provisioning.git

cd cloud-provisioning
scripts/dm-swarm.sh

eval $(docker-machine env swarm-1)

docker node ls

```
The script joined all 3 nodes as managers to the swarm cluster. As a rule of thumb, we should always have at least 3 swarm managers for HA.

### Setting up Consul as Service Registry

Install consul server on swarm-1 node and consul agents on nodes 2 and 3.

```
curl -o docker-compose-proxy.yml \
    https://raw.githubusercontent.com/\
vfarcic/docker-flow-proxy/master/docker-compose.yml

export DOCKER_IP=$(docker-machine ip swarm-1)

docker-compose -f docker-compose-proxy.yml \
    up -d consul-server

```

Test the consul server by writing key value pair and reading it:

```
curl -X PUT -d 'this is a test' \
    http://$(docker-machine ip swarm-1):8500/v1/kv/msg1
    
curl http://$(docker-machine ip swarm-1):8500/v1/kv/msg1?raw
```

Install consul agents (for fault tolerance) on the other 2 nodes:

```
export CONSUL_SERVER_IP=$(docker-machine ip swarm-1)

for i in 2 3; do
    eval $(docker-machine env swarm-$i)

    export DOCKER_IP=$(docker-machine ip swarm-$i)

    docker-compose -f docker-compose-proxy.yml \
        up -d consul-agent
done

```

Consul uses the gossip protocol. As long as every instance is aware of at least one other instance, the protocol will propagate the information across all of them.

We can confirm gossip protocol is working by now trying to get the msg1 from swarm-2 server.

curl http://$(docker-machine ip swarm-2):8500/v1/kv/msg1

## Challenges in scaling stateful services
1. Propagating change in state of one instance to all other instance replicas
2. Creating a copy of a service instance such that its state is retained in the new replica

###Example: Docker-Flow-Proxy design

Create the Docker flow proxy service

```
docker network create --driver overlay proxy

docker service create --name proxy \
    -p 80:80 \
    -p 443:443 \
    -p 8080:8080 \
    --network proxy \
    -e MODE=swarm \
    --replicas 3 \
    -e CONSUL_ADDRESS="$(docker-machine ip swarm-1):8500,$(docker-machine ip swarm-2):8500,$(docker-machine ip swarm-3):8500" \
    vfarcic/docker-flow-proxy

docker service ps proxy
```
The proxy is made in a way that it will try the first address. If it does not respond, it will try the next one, and so on. That way, as long as at least one Consul instance is running, the proxy will be able to fetch and put data. This would not be needed if Consul could run as a swarm service (which is not the case at present). If consul were a swarm service all we needed to do was create an overlay network between consul service and the proxy service and proxy could then have used the Consul service to store its configuration.

Re-create the go-demo service to which the reverse proxy will route the requests to.

```
docker network create --driver overlay go-demo

docker service create --name go-demo-db \
    --network go-demo \
    mongo:3.2.10

docker service create --name go-demo \
    -e DB=go-demo-db \
    --network go-demo \
    --network proxy \
    vfarcic/go-demo:1.0

curl "$(docker-machine ip swarm-1):8080/v1/docker-flow-proxy/reconfigure?serviceName=go-demo&servicePath=/demo&port=8080&distribute=true"

curl -i $(docker-machine ip swarm-1)/demo/hello

```
Every time an instance of the proxy receives a request that results in the change of its state, it propagates that change to other instances, as well as to Consul. On the other hand, the first action the proxy performs when initialized is to consult Consul, and create the configuration from its data.

Swarm will detect that an instance crashed and schedule a new one. The new instance will repeat the same process of querying Consul and create the same state as the other instances.

To verify this scale the proxy to 6 nodes and verify the haproxy.cfg of the new proxy.6 node. We will find that it was able to get the existing configuration of routing /go-demo messages to go-demo service from Consul service.

```
docker service scale proxy=6

NODE=$(docker service ps proxy | grep "proxy.6" | awk '{print $4}')

eval $(docker-machine env $NODE)

ID=$(docker ps | grep "proxy.6" | awk '{print $1}')

docker exec -it $ID cat /cfg/haproxy.cfg

```

# Chapter 5 - Continuous Delivery with Docker

Continuous delivery is a process applied to every commit in a code repository and results in every successful build being ready for deployment to production.

Continuous deployment is a process applied to every commit in a code repository and results with every successful build being deployed to production.

scripts/dm-swarm.sh

eval $(docker-machine env swarm-1)

docker node ls

