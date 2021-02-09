# RedPanda studies

![redpanda](docs/redpanda.png) [RedPanda]() is a modern streaming platform for mission critical workloads, and is compatible with Kafka API. It is a cluster of brokers without any zookeepers.

* Companies are not happy to run Kafka, and when they have deep knowledge, they struggle with performance
* RedPanda scales more easily and delivers consistently low latencies with a much lower hardware footprint, and with data safety.
* To get strong guarantee in Kafka you need to flush message to disk, and it is too slow.
* JVM app tends to have some latency issue while GC (2 to 6s), but with Kafka messaging that will impact throughput too. An asynch write on a file, will put some performance barrier on the OS Kernel, code has to wait the flush to disk to complete. RedPanda uses batch and debouncing the write operations. No page cached is used, so no lock is used on the kernel (file handlers) to save to dick. File space and metadata are preallocated: No synchronization of the file metadata at the linux kernel.
* Result 4x throughput and 100x latency improvements
* The autotuner feature is a tool to assess what the best settings and configurations for running the system: the linux kernel is optimized: network interface, Non-volatile Memory express device (NVMe)

It uses the [Raft consensus algorithm](https://raft.github.io/): recall that **consensus** involves multiple servers agreeing on values. The algorithm is well [explained here](http://thesecretlivesofdata.com/raft/) and can be summarized as:

  * a node has 3 states: Follower, Candidate or Leader
  * all nodes starts as a follower. But if they do not get info from a Leader they can become a candidate
  * candidate requests votes from other nodes
  * candidate becomes the leader if it gets votes from a majority of nodes. Requiring a majority of votes guarantees that only one leader can be elected per term.
  * the leader is getting write/ read operations
  * node uses log to keep command of what to do, it pushes replicas to followers, once a majority of nodes have written the entry, then it commits the change, and then notifies the followers that the entry is committed.
  * The cluster has now come to consensus about the system state.
  * For leader election uses 2 timers:
     
     * election timeout (150ms to 300ms) is the amount of time a follower waits until becoming a candidate. Once it votes for itself, it sends Request Vote messages to other nodes.
     * Heartbeat timeout, controls how often Append Entry messages are sent to followers. Data received from external clients are carried in the Append Entry messages.
  * Raft can stay consistent in the face of network partitions

Is is C++ based.

It supports WASM ([WebAssembly](https://webassembly.org/)) engine to do inline message transformation between topics via uploading WASM script. The Kafka Stream pattern of *consume - process - produce* is using network communication that is not efficient.

Shadow indexing feature helps to upload the append log to long persistence storage like s3, COS with indexing. No need for mirror maker. Ask s3 to do replication between DCs, but index is also copied, so historical access is kept. RedPanda guarantee the access and encapsulate that, still using Kafka API. Super cheap using s3 replication. 

Focus is the company who are doing large scale data streaming without data loss.

## The Kafka API

Kafka API, and Kafka streams are the winner for integrating and processing data streams. Has millions of line of code. Just change the backbone.

RedPanda guarantees compatibility.

REST request / response is supported with Kafka REST proxy, and RedPanda offers the same capability. With this API you can replace MQ based product.

Kafka API had knowledge of the cluster, bootstrap servers, broker lists, partition leader...

But per say Kafka Stream is not that efficient as consume - processing - produce is using network communication, so this transformation can be done with WASM


## Run it

[Installation instruction for rpk](https://vectorized.io/): `brew install vectorizedio/tap/redpanda`

```shell
rpk container start -n 3 
```

Create topic: `rpk topic create test -p 1 -r 1`
Consume from topic: `rpk topic consume test`


Another way to start is to use: `docker run -ti -p 9092:9092 vectorized/redpanda`

### OpenShift Deploy:

* See [deploy app with Helm 3](https://www.openshift.com/blog/openshift-4-3-deploy-applications-with-helm-3):

 ```shell
 curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-darwin-amd64 -o /usr/local/bin/helm
 chmod +x /usr/local/bin/helm
 helm version
 ```

* Create redpanda project: `oc new-project redpanda`
* Get the [Helm Chart](https://github.com/vectorizedio/helm-charts) and modify some values in the values.yaml

  ```shell
  git clone git@github.com:vectorizedio/helm-charts.git
  ```

* update values.yaml to modify the security context and specify the storage class

 ```yaml
 serviceAccount:
  create: true
  annotations: {}
  name: "redpanda-sa"
  
 podSecurityContext: {}
 securityContext: 
    readOnlyRootFilesystem: true
    runAsNonRoot: true
 storage:   
   storageClass: "ibmc-block-gold"
 ```

* Install

  ```shell
  helm install --namespace redpanda redpanda ./redpanda/
  ```

The output looks like:

```shell
NAME: redpanda
LAST DEPLOYED: Mon Feb  8 16:49:18 2021
NAMESPACE: redpanda
STATUS: deployed
REVISION: 1
NOTES:
Congratulations on installing redpanda!

The pods will rollout in a few seconds. To check the status:

  kubectl -n redpanda rollout status -w statefulset/redpanda
```

* Try some sample commands, like creating a topic called topic1:

  ```shell
  oc  run -ti --rm --restart=Never \
    --image vectorized/redpanda:latest \
    rpk -- --brokers=redpanda-bootstrap:9092 api topic create topic1
  ```

* To get the api status:
 
 ```shell
  kubectl -n redpanda run -ti --rm --restart=Never \
    --image vectorized/redpanda:latest \
    rpk -- --brokers=redpanda-bootstrap:9092 api status

```

* Get services: `oc get svc` and then expose the bootstrap as route: `oc expose svc redpanda-bootstrap`

## Compendium

* [Podcast - simplify your streaming data workloads with Red Panda.](https://www.dataengineeringpodcast.com/vectorized-red-panda-streaming-data-episode-152/)
* [Performance summit 2020](https://www.youtube.com/watch?v=wwU58YMgPtE&t=1944s)
* [Helm chart]()