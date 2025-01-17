# Strimzi mirroring with MirrorMaker2 and Submariner

## Introduction
This repository aims to show how it's possible to configure a couple of kafka clusters each of them in a different k8s cluster to have a mirroring process from one to another kafka installation using MirrorMaker2 as mirror manager and Submariner as networking infrastructure to enable direct and secure networking between the k8s clusters.

This scenario could be used in a lot of [use cases](https://developers.redhat.com/articles/2023/11/13/demystifying-kafka-mirrormaker-2-use-cases-and-architecture#use_cases):
* DR (Disaster Recovery) 
* Data isolation
* Data aggregation
* Data migration

## Create K8s clusters with Kind
Create clusters:

```bash
kind create cluster --config primary/kind.yaml
kind create cluster --config backup/kind.yaml
```

Here we can notice that we have choosen two networks for pods and services not overlapped between clusters. 

For the primary clutser:
```bash
networking:
  podSubnet: 10.240.0.0/16
  serviceSubnet: 10.110.0.0/16
```

For the backup clutser:
```bash
networking:
  podSubnet: 10.241.0.0/16
  serviceSubnet: 10.111.0.0/16
```

Pod CIDRs '10.240.0.0/16' and '10.241.0.0/16' are not overlapped each other.
Same thing happens for service CIDRs '10.110.0.0/16' and '10.111.0.0/16'.
In such case we can use submariner without enabling its Globalnet feature.

Now we can start installing Submariner.

## Install Submariner

### Install Submariner Broker

We want to install submariner broker into backup cluster in its control-plane node using the submariner CLI subctl.

Before to proceed with that installation I have to retrieve the internal IP address of both cluster control-plane nodes. This addresses are needed to configure a valid kube-context which the subctl can use to install submariner components in both clusters.

```bash
export PRIMARY_INTERNAL_IPADDRESS=$(kubectl --context kind-primary get nodes primary-control-plane -o jsonpath="{.status.addresses[?(@.type=='InternalIP')].address}")
export BACKUP_INTERNAL_IPADDRESS=$(kubectl --context kind-backup get nodes backup-control-plane -o jsonpath="{.status.addresses[?(@.type=='InternalIP')].address}")
```

Copy the local kubeconfig file to backup control-plane node where we want to install the submariner broker:

```bash
docker cp ~/.kube/config backup-control-plane:/root/.kube/config
```

Open a terminal in backup-control-plane container hosting the control-plane node of backup cluster and set there the environment variables relative to the internal ip addresses previously discovered

```bash
docker exec -it --env PRIMARY_INTERNAL_IPADDRESS --env BACKUP_INTERNAL_IPADDRESS backup-control-plane /bin/bash
```

Let’s use those environment variables (that we passed into the container) to set the correct IP Addresses in the kubeconfig file previosly copied there. Before that we have to unset the environment variable KUBECONFIG so that kubectl can auromatically consider the kubeconfig file in ${HOME}/.kube/config

```bash
unset KUBECONFIG
kubectl config set-cluster kind-primary --server https://$PRIMARY_INTERNAL_IPADDRESS:6443
kubectl config set-cluster kind-backup --server https://$BACKUP_INTERNAL_IPADDRESS:6443
```

Install the subctl binary

```bash
curl -Ls https://get.submariner.io | bash
export PATH=$PATH:/root/.local/bin
```

Install submariner broker

```bash
cd /root
subctl deploy-broker
```

you should to see this

```bash
root@backup-control-plane:/# subctl deploy-broker
 ✓ Setting up broker RBAC
 ✓ Deploying the Submariner operator
 ✓ Created operator CRDs
 ✓ Created operator namespace: submariner-operator
 ✓ Created operator service account and role
 ✓ Created submariner service account and role
 ✓ Created lighthouse service account and role
 ✓ Deployed the operator successfully
 ✓ Deploying the broker
 ✓ Saving broker info to file "broker-info.subm"
```

### Install Submariner Gateway Engine

Using a local terminal, add the submariner label to select some worker nodes to install the submariner gateway in both clusters (we will use primary-worker node of primary cluster and backup-worker node of backup cluster)

```bash
kubectl --context kind-primary label node primary-worker submariner.io/gateway=true
kubectl --context kind-backup label node backup-worker submariner.io/gateway=true
```

coming back to the terminal opened into backup-control-plane container, use suctl to install a submariner gateway in each cluster:


```bash
subctl join --context kind-backup broker-info.subm --clusterid kind-backup --natt=false --cable-driver=vxlan
subctl join --context kind-primary broker-info.subm --clusterid kind-primary --natt=false --cable-driver=vxlan
```

Here we can notice that we have used vxlan as cable-driver so that between the submariner cluster gateways (one per node) we are not going to instantiate a IPSEC tunnel (default or --cable-driver=libreswan), but an un-encrypted tunnel implementation based on VXLAN. This kind of connection is the same used by submariner to enable the communications intra-cluster between the route agents (one per node) and the gateway. Here we had to use this kind of tunnelling beetween gateways because the actual version of library libreswan needs to access to some kernel configurations of the cluster node but not available in the container node istanziated by Kind. Obviously, when it's possible, it's always better to use IPSEC tunnelling for the communication between two submariner gateways for security reasons. 

After that we can check pod status in submariner-operator namespace in both clusters using our local terminal

```bash
kubectl --context kind-primary get pod -n submariner-operator -w
```

```bash
kubectl --context kind-backup get pod -n submariner-operator -w
```

When everything will be OK, we can check the Submariner status invoking in the terminal opened into backup-control-plane container:

```bash
subctl diagnose all
```

You should see something like that

```bash
Cluster "kind-primary"
 ✓ Checking Submariner support for the Kubernetes version
 ✓ Kubernetes version "v1.32.0" is supported

 ✓ Non-Globalnet deployment detected - checking that cluster CIDRs do not overlap
 ✓ Checking DaemonSet "submariner-gateway"
 ✓ Checking DaemonSet "submariner-routeagent"
 ✓ Checking DaemonSet "submariner-metrics-proxy"
 ✓ Checking Deployment "submariner-lighthouse-agent"
 ✓ Checking Deployment "submariner-lighthouse-coredns"
 ✓ Checking the status of all Submariner pods
 ✓ Checking that gateway metrics are accessible from non-gateway nodes

 ✓ Checking Submariner support for the CNI network plugin
 ✓ The detected CNI network plugin ("kindnet") is supported
 ✓ Checking gateway connections
 ✓ Checking route agent connections
 ✓ Checking Submariner support for the kube-proxy mode
 ✓ The kube-proxy mode is supported
 ✓ Checking that firewall configuration allows intra-cluster VXLAN traffic

 ✓ Checking that services have been exported properly


Cluster "kind-backup"
 ✓ Checking Submariner support for the Kubernetes version
 ✓ Kubernetes version "v1.32.0" is supported

 ✓ Non-Globalnet deployment detected - checking that cluster CIDRs do not overlap
 ✓ Checking DaemonSet "submariner-gateway"
 ✓ Checking DaemonSet "submariner-routeagent"
 ✓ Checking DaemonSet "submariner-metrics-proxy"
 ✓ Checking Deployment "submariner-lighthouse-agent"
 ✓ Checking Deployment "submariner-lighthouse-coredns"
 ✓ Checking the status of all Submariner pods
 ✓ Checking that gateway metrics are accessible from non-gateway nodes

 ✓ Checking Submariner support for the CNI network plugin
 ✓ The detected CNI network plugin ("kindnet") is supported
 ✓ Checking gateway connections
 ✓ Checking route agent connections
 ✓ Checking Submariner support for the kube-proxy mode
 ✓ The kube-proxy mode is supported
 ✓ Checking that firewall configuration allows intra-cluster VXLAN traffic

 ✓ Checking that services have been exported properly
```

If we want to get some info about submariner and his components in our clusters, we can 
using this command

```bash
subctl show all
```

You should see something like that

```bash
Cluster "kind-backup"
 ✓ Detecting broker(s)
NAMESPACE               NAME                COMPONENTS                        GLOBALNET   GLOBALNET CIDR   DEFAULT GLOBALNET SIZE   DEFAULT DOMAINS
submariner-k8s-broker   submariner-broker   service-discovery, connectivity   no          242.0.0.0/8      65536

 ✓ Showing Connections
GATEWAY          CLUSTER        REMOTE IP    NAT   CABLE DRIVER   SUBNETS                        STATUS      RTT avg.
primary-worker   kind-primary   10.89.1.37   no    vxlan          10.110.0.0/16, 10.240.0.0/16   connected   430.943µs

 ✓ Showing Endpoints
CLUSTER        ENDPOINT IP   PUBLIC IP     CABLE DRIVER   TYPE
kind-backup    10.89.1.39    93.66.84.49   vxlan          local
kind-primary   10.89.1.37    93.66.84.49   vxlan          remote

 ✓ Showing Gateways
NODE            HA STATUS   SUMMARY
backup-worker   active      All connections (1) are established

 ✓ Showing Network details
    Discovered network details via Submariner:
        Network plugin:  kindnet
        Service CIDRs:   [10.111.0.0/16]
        Cluster CIDRs:   [10.241.0.0/16]
        ClustersetIP CIDR:     243.0.0.0/20

 ✓ Showing versions
COMPONENT                       REPOSITORY           CONFIGURED   RUNNING                     ARCH
submariner-gateway              quay.io/submariner   0.19.2       release-0.19-0b83248e63f1   arm64
submariner-routeagent           quay.io/submariner   0.19.2       release-0.19-0b83248e63f1   arm64
submariner-metrics-proxy        quay.io/submariner   0.19.2       Unavailable                 Unavailable
submariner-operator             quay.io/submariner   0.19.2       release-0.19-87a5522dbead   arm64
submariner-lighthouse-agent     quay.io/submariner   0.19.2       release-0.19-c138ebf9dfe9   arm64
submariner-lighthouse-coredns   quay.io/submariner   0.19.2       release-0.19-c138ebf9dfe9   arm64


Cluster "kind-primary"
 ✓ Detecting broker(s)
 ✓ No brokers found

 ✓ Showing Connections
GATEWAY         CLUSTER       REMOTE IP    NAT   CABLE DRIVER   SUBNETS                        STATUS      RTT avg.
backup-worker   kind-backup   10.89.1.39   no    vxlan          10.111.0.0/16, 10.241.0.0/16   connected   439.279µs

 ✓ Showing Endpoints
CLUSTER        ENDPOINT IP   PUBLIC IP     CABLE DRIVER   TYPE
kind-primary   10.89.1.37    93.66.84.49   vxlan          local
kind-backup    10.89.1.39    93.66.84.49   vxlan          remote

 ✓ Showing Gateways
NODE             HA STATUS   SUMMARY
primary-worker   active      All connections (1) are established

 ✓ Showing Network details
    Discovered network details via Submariner:
        Network plugin:  kindnet
        Service CIDRs:   [10.110.0.0/16]
        Cluster CIDRs:   [10.240.0.0/16]
        ClustersetIP CIDR:     243.0.16.0/20

 ✓ Showing versions
COMPONENT                       REPOSITORY           CONFIGURED   RUNNING                     ARCH
submariner-gateway              quay.io/submariner   0.19.2       release-0.19-0b83248e63f1   arm64
submariner-routeagent           quay.io/submariner   0.19.2       release-0.19-0b83248e63f1   arm64
submariner-metrics-proxy        quay.io/submariner   0.19.2       Unavailable                 Unavailable
submariner-operator             quay.io/submariner   0.19.2       release-0.19-87a5522dbead   arm64
submariner-lighthouse-agent     quay.io/submariner   0.19.2       release-0.19-c138ebf9dfe9   arm64
submariner-lighthouse-coredns   quay.io/submariner   0.19.2       release-0.19-c138ebf9dfe9   arm64
```

### Test submariner export service (aka service discovery) between clusters

Create the same namespace in both cluters

```bash
kubectl --context kind-primary create namespace test
kubectl --context kind-backup create namespace test
```

Create a nginx deploy in backup cluster and expose it through a service

```bash
kubectl --context kind-backup -n test create deployment nginx --image=nginx
kubectl --context kind-backup -n test expose deployment nginx --port=80
```

Export the nginx service using submariner so that it is available in both cluster in the domain clusterset.local 

```bash
subctl export service --context kind-backup --namespace test nginx
```

Deploy a nettest container in the primary cluster and open a in it

```bash
kubectl --context kind-primary run -n default tmp-shell --rm -i --tty --image quay.io/submariner/nettest -- /bin/bash
```

Try to invoke the nginx service in the backup cluster simply executing a curl in the previolsy opened terminal

```bash
curl nginx.test.svc.clusterset.local
```

You should see the welcome page of nginx server

```bash
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Install Kakfa and enable mirroring

### Install Strimzi operator

Before deploying the Strimzi cluster operator, create a namespace called kafka where we are going to install strimzi operator and then a kafka cluster:

```bash
kubectl --context kind-primary create namespace kafka
kubectl --context kind-backup create namespace kafka
```

Apply the Strimzi install files including ClusterRoles, ClusterRoleBindings, some Custom Resource Definitions (CRDs) and operator deployment:

```bash
kubectl --context kind-primary create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
kubectl --context kind-backup create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Follow the deployment of the Strimzi cluster operator in both clusters in two different shells:
```bash
kubectl --context kind-primary get pod -n kafka --watch -o wide
```

```bash
kubectl --context kind-backup get pod -n kafka --watch -o wide
```

### Deploy Kafka clusters

Create the primary kafka cluster and wait it becomes ready:

```bash
kubectl --context kind-primary apply -f primary/kafka.yaml -n kafka
kubectl --context kind-primary wait kafka/streams-cluster --for=condition=Ready --timeout=500s -n kafka
```

Create the backup kafka cluster and wait it becomes ready:

```bash
kubectl --context kind-backup apply -f backup/kafka.yaml -n kafka
kubectl --context kind-backup wait kafka/streams-cluster-backup --for=condition=Ready --timeout=500s -n kafka
```

For the moment the created kafka cluster are independent each other. The resource configuration is the same for both cluster with the exception of the cluster name which is rispectively:
* streams-cluster
* streams-cluster-backup

In the cluster configuration there is an important thing we have to notice: in each cluster we have created two listeners:
* a listener named 'plain' and type 'internal' that we imagine using it to enable consumers and producers to read and write from/to kafka topics;
* a listener named 'mirroring' and type 'cluster-ip' that we imagine using it to enable the mirroring flow between the kafka clusters;

We are going to install the MirrorMaker2 in the backup cluster to manage the mirroring flow from the source kafka cluster in the primary k8s cluster and the target kafka cluster in the same backup k8s cluster. We have choosen to use the type 'cluster-ip' for the mirroring listener because in this way we'll the operator will create a bootstrap service and a service foreach kafka brokers dedicated to the mirroring listener and we'll be able to export these services using submariner so that they can be usable from any k8s cluster. We'll see shortly how this thing is at the base of kafka mirroring using submariner.

### Create the topics

Now we are going to create a topic in the primary kafka cluster and the relative topic (with the same name and configuration) in the backup kafka cluster. This second topic will be the destination of mirrored messages from the first topic by MirrorMaker2. MM2 could create the destination topic automatically, but in same case this behaviour can produce some mistackes in the topic configuration. For this reason we'll prefer to disable this MM2 behaviour in its resource and create manually the destination topic.

Create the topic into the primary cluster and check it becomes ready:
```bash
kubectl --context kind-primary apply -f primary/topic.yaml -n kafka
kubectl --context kind-primary wait kafkatopic/my-topic --for=condition=Ready --timeout=300s -n kafka
```

Create the topic into the backup cluster and check it becomes ready:
```bash
kubectl --context kind-backup apply -f backup/topic.yaml -n kafka
kubectl --context kind-backup wait kafkatopic/my-topic --for=condition=Ready --timeout=300s -n kafka
```

### Create the kafka users

Here we are going to create a couple of kafka user for each cluster. In particluar we'll create:
* in the primary kafka cluster:
  * a user with writing permissions to the topic created previosly
  * a user for mirroring with admin permissions
* in the backup kafka cluster:
  * a user with reading permissions to the topic created previosly
  * a user for mirroring with admin permissions

In our scenario the main role will be played by two mirroring users. MM2 will use them to read and write mirrored data from the primary cluster to the backup cluster. The other users will be used by a producer and consumer client into the primary and backap cluster rispectively.

Create the writer user into the primary cluster (mm2 source cluster) and wait for it to be ready:

```bash
kubectl --context kind-primary apply -f primary/writer-user.yaml -n kafka
kubectl --context kind-primary wait kafkauser/writer-user --for=condition=Ready --timeout=300s -n kafka
```

Create the primary mirroring user into the primary cluster (mm2 source cluster) and wait for it to be ready:

```bash
kubectl --context kind-primary apply -f primary/primary-mirroring-user.yaml -n kafka
kubectl --context kind-primary wait kafkauser/primary-mirroring-user --for=condition=Ready --timeout=300s -n kafka
```

Create the reader user into the backup cluster (mm2 target cluster) and wait for it to be ready:

```bash
kubectl --context kind-backup apply -f backup/reader-user.yaml -n kafka
kubectl --context kind-backup wait kafkauser/reader-user --for=condition=Ready --timeout=300s -n kafka
```

Create the backup mirroring user into the backup cluster (mm2 target cluster) and wait for it to be ready:

```bash
kubectl --context kind-backup apply -f backup/backup-mirroring-user.yaml -n kafka
kubectl --context kind-backup wait kafkauser/backup-mirroring-user --for=condition=Ready --timeout=300s -n kafka
```

### Export the primary mirroring user secret to the backup cluster 

How we said the MM2 will be installed in the backup cluster and it needs both kafka mirroring users to read from primary cluster and write to backup cluster. Now MM2 can access to the backup mirroring user credential secret directly (they are in the same cluster), but this is not equal for the primary mirroring user credential secret. For this reason we create a copy of that secret in the backup cluster using:

```bash
export PRIMARY_MIRRORING_USER_PASSWORD=$(kubectl --context kind-primary get secret primary-mirroring-user -n kafka -o jsonpath='{.data.password}' | base64 -d)
kubectl --context kind-backup create secret generic primary-mirroring-user -n kafka --from-literal=password=$PRIMARY_MIRRORING_USER_PASSWORD
```

### Export the primary kafka mirroring services using submariner

If you have the submariner CLI binary installed in your local machine you can execute this commands in a local shell, otherwise you can use the same terminal opened in the backup-control-plain container previusly when we installed submariner components:

```bash
subctl export service --context kind-primary --namespace kafka streams-cluster-kafka-mirroring-bootstrap
subctl export service --context kind-primary --namespace kafka streams-cluster-kafka-mirroring-0
subctl export service --context kind-primary --namespace kafka streams-cluster-kafka-mirroring-1
subctl export service --context kind-primary --namespace kafka streams-cluster-kafka-mirroring-2
```

After that the mirroring kafka bootstrap service and services foreach kafka brokers of the primary cluster can be resolved in both k8s clusters in the domain 'clusterset.local' using the hostnames:

* streams-cluster-kafka-mirroring-bootstrap.kafka.svc.clusterset.local
* streams-cluster-kafka-mirroring-0.kafka.svc.clusterset.local
* streams-cluster-kafka-mirroring-1.kafka.svc.clusterset.local
* streams-cluster-kafka-mirroring-2.kafka.svc.clusterset.local

### Deploy MirrorMaker2 (MM2) in the backup cluster

Create the mm2 pod (in production environments increase its replicas) in the backup cluster to manage kafka mirroring from primary to backup kafka clusters and wait it to be ready:

```bash
kubectl --context kind-backup apply -f backup/mm2.yaml -n kafka
kubectl wait kafkamirrormaker2/mirror-maker-2-backup --for=condition=Ready --timeout=300s -n kafka
```

There are a lot of thing to notice in the created mm2 resource. The most important things are:
* in the clusters section
  * for the source cluster
    * the bootstrapServers field with the value 'streams-cluster-kafka-mirroring-bootstrap.kafka.svc.clusterset local:9093'. It's the hostname and port of mirroring bootstrap service of the primary kafka cluster exported via submariner. This service will give MM2 client its list of mirroring kafka broker services and, because they are available from backup cluster (where mm2 is deployed), we have specified in the mirroring listener of primary kafka cluster the advertised host of each broker using the hostnames in the domain 'clusterset.local' (the domain resolved by submariner DNS services).
      ```bash
        - name: mirroring
        port: 9093
        type: cluster-ip
        tls: false
        authentication:
          type: scram-sha-512
        configuration:
          brokers:
          - broker: 0
            advertisedHost: streams-cluster-kafka-mirroring-0.kafka.svc.clusterset.local
          - broker: 1
            advertisedHost: streams-cluster-kafka-mirroring-1.kafka.svc.clusterset.local
          - broker: 2
            advertisedHost: streams-cluster-kafka-mirroring-2.kafka.svc.clusterset.local
      ```
     * the authentication section where is configured the reference to the primary mirroring user secret that we have previously copied in the backup from the primary cluster.
  * for the target cluster
    * the bootstrapServers field with the value 'streams-cluster-backup-kafka-mirroring-bootstrap.kafka.svc.cluster.local:9093'. It's the hostname and port of mirroring bootstrap service of the backup kafka cluster. Notice how we can use the service FQDN in the 'cluster.local' domain (the domain resolved by k8s DNS services).
     * the authentication section where is configured the reference to the backup mirroring user secret.
* in the mirrors section 
  * for the connector sections
    * 'sync.topic.configs.enabled: "false"', 'refresh.topics.enabled: "false"' and 'topic.creation.enable: "false"' configs related to our decision to disable mm2 topic creation feature in favor of manually topic creation
    * 'replication.policy.class: "io.strimzi.kafka.connect.mirror.IdentityReplicationPolicy"' config to mirror the data from a topic in the source cluster to another topic in the target but with the same name. The default replication policy uses for the target topic the name of the source topic prefixed with the name of the source cluster.
  * the topicsPattern and groupsPattern fields where it's possibe to use regex or comma-separated list of the topics/groups managed by mirroring process.


## Test the mirroring

At this point we can test our kafka mirroring solution with NMM2 and Submriner.

We are going to create a kafka producer client writing on a topic of the primary kafka cluster and a kafka consumer client reading from the topic with the same name (and under mirroring process) of the backup kafka cluster.

Sample producer authenticated with the writer user and attached to the plain listener of primary kafka cluster:
```bash
export WRITER_USER_PASSWORD=$(kubectl --context kind-primary get secret writer-user -n kafka -o jsonpath='{.data.password}' | base64 -d)
kubectl --context kind-primary run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka -- /bin/bash -c "cat >/tmp/producer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=writer-user password=$WRITER_USER_PASSWORD;
EOF
bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic --producer.config=/tmp/producer.properties"
```

Sample consumer authenticated with the reader user and attached to the plain listener of backup kafka cluster:
```bash
export READER_USER_PASSWORD=$(kubectl --context kind-backup get secret reader-user -n kafka -o jsonpath='{.data.password}' | base64 -d)
kubectl --context kind-backup run kafka-consumer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.9.0 --rm=true --restart=Never -n kafka -- /bin/bash -c "cat >/tmp/consumer.properties <<EOF 
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username=reader-user password=$READER_USER_PASSWORD;
EOF
bin/kafka-console-consumer.sh --bootstrap-server streams-cluster-backup-kafka-bootstrap:9092 --topic my-topic --consumer.config=/tmp/consumer.properties --group my-group"
```

If all is ok, writing some messages in the producer terminal, we should see after a bit time the same messages in the consumer terminal.

## References
Projects
* [Kind](https://kind.sigs.k8s.io/)
* [Submariner](https://submariner.io/)
* [Strimzi](https://strimzi.io/documentation/)
* [Strimzi API Reference](https://strimzi.io/docs/operators/latest/configuring)
Articles
* [Demystifying Kafka MirrorMaker 2: Use cases and architecture](https://developers.redhat.com/articles/2023/11/13/demystifying-kafka-mirrormaker-2-use-cases-and-architecture#)
* [Mastering Kafka migration with MirrorMaker 2](https://developers.redhat.com/articles/2024/01/04/mastering-kafka-migration-mirrormaker-2#)
* [How to use MirrorMaker 2 with OpenShift Streams for Apache Kafka](https://developers.redhat.com/articles/2021/12/15/how-use-mirrormaker-2-openshift-streams-apache-kafka#)
Github
* [rmarting/strimzi-migration-demo](https://github.com/rmarting/strimzi-migration-demo/tree/main)