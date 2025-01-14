# Strimzi mirroring
This repository aim to provide an example Kafka CR for deploying a kafka cluster across multiple AZ/DC with dedicated nodes and rack-awareness.

The rack awareness feature spreads replicas of the same partition across different racks. This extends the guarantees Kafka provides for broker-failure to cover rack-failure, limiting the risk of data loss should all the brokers on a rack fail at once. The feature can also be applied to other broker groupings such as availability zones in EC2.

For further details see the following links:
- [strimzi-running-kafka-on-dedicated-nodes](https://strimzi.io/blog/2018/07/30/running-kafka-on-dedicated-nodes/)
- [strimzi-spreading-partition-replicas-across-racks](https://github.com/strimzi/strimzi-kafka-operator/blob/main/documentation/api/io.strimzi.api.kafka.model.common.Rack.adoc)
  
## Create K8s clusters
Create clusters:
```bash
kind create cluster --config primary/kind.yaml
kind create cluster --config backup/kind.yaml
```

To create a dedicated node, you have to first taint it. Taints are a feature of nodes which can be used to repel pods. Only pods which tolerate a given taint can be scheduled on such nodes. The nodes can be tainted using the kubectl tool:
```bash
kubectl taint nodes k8s-cluster-worker2 dedicated=primary:NoSchedule
kubectl taint nodes k8s-cluster-worker3 dedicated=backup:NoSchedule
```

Verify nodes labels:
```bash
kubectl get nodes --show-labels=true
```

Output:
```
NAME                        STATUS   ROLES           AGE   VERSION   LABELS
k8s-cluster-control-plane   Ready    control-plane   85s   v1.32.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=k8s-cluster-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
k8s-cluster-worker          Ready    <none>          72s   v1.32.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,kubernetes.io/arch=arm64,kubernetes.io/hostname=k8s-cluster-worker,kubernetes.io/os=linux
k8s-cluster-worker2         Ready    <none>          73s   v1.32.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,dedicated=primary,kubernetes.io/arch=arm64,kubernetes.io/hostname=k8s-cluster-worker2,kubernetes.io/os=linux
k8s-cluster-worker3         Ready    <none>          73s   v1.32.0   beta.kubernetes.io/arch=arm64,beta.kubernetes.io/os=linux,dedicated=backup,kubernetes.io/arch=arm64,kubernetes.io/hostname=k8s-cluster-worker3,kubernetes.io/os=linux
```

Verify nodes taints:
```bash
kubectl describe nodes | grep -E "Name:|Taints:"
```

Output:
```
Name:               k8s-cluster-control-plane
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Name:               k8s-cluster-worker
Taints:             <none>
Name:               k8s-cluster-worker2
Taints:             dedicated=primary:NoSchedule
Name:               k8s-cluster-worker3
Taints:             dedicated=backup:NoSchedule
```

## Install Strimzi operator
Before deploying the Strimzi cluster operator, create a namespace called kafka:
```bash
kubectl create namespace kafka
```

Apply the Strimzi install files, including ClusterRoles, ClusterRoleBindings and some Custom Resource Definitions (CRDs):
```bash
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Follow the deployment of the Strimzi cluster operator:
```bash
kubectl get pod -n kafka --watch -o wide
```

## Deploy Kafka clusters

Create the primary kafka cluster:
```bash
kubectl apply -f primary/kafka.yaml -n kafka
```

Waiting for primary cluster to be ready:
```bash
kubectl wait kafka/streams-cluster --for=condition=Ready --timeout=300s -n kafka
```

Create the backup kafka cluster:
```bash
kubectl apply -f kafka-backup.yaml -n kafka
```

Waiting for backup cluster to be ready:
```bash
kubectl wait kafka/kafka-backup --for=condition=Ready --timeout=300s -n kafka
```

Verify the pod status:
```bash
kubectl get pod -n kafka --watch -o wide
```

Output:
```
NAME                                             READY   STATUS    RESTARTS   AGE     IP           NODE                  NOMINATED NODE   READINESS GATES
kafka-backup-entity-operator-645649f96f-k2tsg    2/2     Running   0          43s     10.244.3.4   k8s-cluster-worker    <none>           <none>
kafka-backup-kafka-0                             1/1     Running   0          70s     10.244.1.5   k8s-cluster-worker3   <none>           <none>
kafka-backup-zookeeper-0                         1/1     Running   0          9m59s   10.244.1.3   k8s-cluster-worker3   <none>           <none>
kafka-primary-entity-operator-6875ff77c4-6gktc   2/2     Running   0          17m     10.244.3.3   k8s-cluster-worker    <none>           <none>
kafka-primary-kafka-0                            1/1     Running   0          18m     10.244.2.5   k8s-cluster-worker2   <none>           <none>
kafka-primary-zookeeper-0                        1/1     Running   0          32m     10.244.2.3   k8s-cluster-worker2   <none>           <none>
strimzi-cluster-operator-76b947897f-gnkvd        1/1     Running   0          49m     10.244.3.2   k8s-cluster-worker    <none>           <none>
```

## Deploy Mirror Maker 2 cluster

Create the mirror maker 2 cluster from primary to backup kafka cluster:
```bash
kubectl apply -f mirrormaker2-backup.yaml -n kafka
```

Waiting for cluster to be ready:
```bash
kubectl wait kafkamirrormaker2/mirror-maker-2-backup --for=condition=Ready --timeout=300s -n kafka
```

Verify the pod status:
```bash
kubectl get pod -n kafka --watch -o wide
```

Output:
```
NAME                                             READY   STATUS    RESTARTS   AGE     IP           NODE                  NOMINATED NODE   READINESS GATES
kafka-backup-entity-operator-645649f96f-k2tsg    2/2     Running   0          32m     10.244.3.4   k8s-cluster-worker    <none>           <none>
kafka-backup-kafka-0                             1/1     Running   0          32m     10.244.1.5   k8s-cluster-worker3   <none>           <none>
kafka-backup-zookeeper-0                         1/1     Running   0          41m     10.244.1.3   k8s-cluster-worker3   <none>           <none>
kafka-primary-entity-operator-6875ff77c4-6gktc   2/2     Running   0          49m     10.244.3.3   k8s-cluster-worker    <none>           <none>
kafka-primary-kafka-0                            1/1     Running   0          49m     10.244.2.5   k8s-cluster-worker2   <none>           <none>
kafka-primary-zookeeper-0                        1/1     Running   0          63m     10.244.2.3   k8s-cluster-worker2   <none>           <none>
mirror-maker-2-backup-mirrormaker2-0             1/1     Running   0          5m54s   10.244.3.5   k8s-cluster-worker    <none>           <none>
strimzi-cluster-operator-76b947897f-gnkvd        1/1     Running   0          81m     10.244.3.2   k8s-cluster-worker    <none>           <none>
```

# Create a topic into the primary cluster 

Create a topic into the primary cluster:
```bash
kubectl apply -f topic.yaml -n kafka
```

Waiting for topic to be ready:
```bash
kubectl wait kafkatopic/my-topic --for=condition=Ready --timeout=300s -n kafka
```

## Send a message to created topic

Create a container to send a message to created topic
```bash
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server kafka-primary-kafka-bootstrap:9092 --topic my-topic
```

Once everything is set up correctly, you’ll see a prompt where you can type in your messages:
```bash
If you don't see a command prompt, try pressing enter.

>Hello
```

kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server kafka-primary-kafka-bootstrap:9092 --topic my-topic --from-beginning

kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server kafka-backup-kafka-bootstrap:9092 --topic my-topic --group pippo

kubectl -n kafka run kafka-test -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server kafka-primary-kafka-bootstrap:9092 --list

kubectl -n kafka run kafka-test -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-topics.sh --bootstrap-server kafka-backup-kafka-bootstrap:9092 --list




kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never

kubectl -n kafka run kafka-consumer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server kafka-backup-kafka-bootstrap:9092 --topic my-topic --from-beginning

kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list kafka-primary-kafka-bootstrap:9092 --topic my-topic | awk -F  ":" '{sum += $3} END {print sum}'
kafka-run-class.sh kafka.admin.ConsumerGroupCommand --group my-group --bootstrap-server kafka-primary-kafka-bootstrap:9092 --topic my-topic --describe



# Install Cloud Provider KIND using golang

go install sigs.k8s.io/cloud-provider-kind@latest

# Running the provider

In base of your local golang configuration the cloud-provider-kind could be in one of these ordered desinations:
$GOBIN
$GOPATH/bin
$HOME/go/bin

Found the binary you can make it available elsewhere:

sudo install <binary_path>/cloud-provider-kind /usr/local/bin

Then, refreshing the shell, execute it using sudo if it's necessary:

[sudo] cloud-provider-kind

For more detals you can check the official documentation at: https://github.com/kubernetes-sigs/cloud-provider-kind?tab=readme-ov-file#install

Sulla mia macchina ho problemi sui container envoy creati dinamicamente da cloud-provider-kind:
cannot bind '0.0.0.0:80': Permission denied
cannot bind '0.0.0.0:443': Permission denied

i problemi sopra lì ho sia che lanci cloud-provider-kind con
sudo cloud-provider-kind
sia con
sudo cloud-provider-kind -enable-lb-port-mapping

Solution: use kind portmapping, ingress-nginx-controller on control-plane node and ingress resources

# Install the Ingress NGINX Controller

kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml

Now the Ingress NGINX Controller is all setup. Wait until is ready to process requests running:

kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s



## Install tigera operator in both clusters
```bash
kubectl --context kind-primary create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl --context kind-backup create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
```

## Install calico in both cluster
```bash
kubectl --context kind-primary apply -f primary/tigera.yaml
kubectl --context kind-backup apply -f backup/tigera.yaml
```

After that we can check pod status in calico-system namespace and node status in both clusters

```bash
kubectl --context kind-primary get pod -n calico-system -w
```

```bash
kubectl --context kind-primary get nodes -w
```

```bash
kubectl --context kind-backup get pod -n calico-system -w
```

```bash
kubectl --context kind-backup get nodes -w
```

If everithing is ok, we can configure Calico IPPools

If you don't have calico CLI locally installed you can see this page https://docs.tigera.io/calico/latest/operations/calicoctl/install#install-calicoctl-as-a-binary-on-a-single-host and follow the its installation instructions (it's better to have a version of calicoctl binary equal or greater than the calico server installed into the clusters).

```bash
calicoctl --context kind-primary create -f primary/ippool.yaml
calicoctl --context kind-backup create -f backup/ippool.yaml
```

Now we can start installing Submariner

## Install Submariner

### Install Submariner Broker

We want to install submariner broker into backup cluster in its control-plane node using the submariner CLI subctl

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

touch /etc/apt/sources.list.d/sources.list
echo "deb http://deb.debian.org/debian bookworm-backports main" >> /etc/apt/sources.list.d/sources.list
apt update
apt-get update
apt install wireguard

coming back to the terminal opened into backup-control-plane container, use suctl to install a submariner gateway in each cluster


```bash
subctl join --context kind-backup broker-info.subm --clusterid kind-backup --natt=false --cable-driver=vxlan --health-check=false
subctl join --context kind-primary broker-info.subm --clusterid kind-primary --natt=false --cable-driver=vxlan --health-check=false
```

After that we can check pod status in submariner-operator namespace in both clusters using our local terminal

```bash
kubectl --context kind-primary get pod -n submariner-operator -w
```

```bash
kubectl --context kind-backup get pod -n submariner-operator -w
```

When everything will be OK, we can check the Submariner status using this command always in the terminal opened into backup-control-plane container

```bash
subctl show all
```

You should see something like that

```bash
Cluster "kind-primary"
 ✓ Detecting broker(s)
 ✓ No brokers found

 ✓ Showing Connections
GATEWAY         CLUSTER       REMOTE IP    NAT   CABLE DRIVER   SUBNETS                        STATUS      RTT avg.
backup-worker   kind-backup   10.89.1.75   no    vxlan          10.111.0.0/16, 10.241.0.0/16   connected

 ✓ Showing Endpoints
CLUSTER        ENDPOINT IP   PUBLIC IP     CABLE DRIVER   TYPE
kind-primary   10.89.1.71    93.66.84.49   vxlan          local
kind-backup    10.89.1.75    93.66.84.49   vxlan          remote

 ✓ Showing Gateways
NODE             HA STATUS   SUMMARY
primary-worker   active      All connections (1) are established

 ✓ Showing Network details
    Discovered network details via Submariner:
        Network plugin:  calico
        Service CIDRs:   [10.110.0.0/16]
        Cluster CIDRs:   [10.240.0.0/16]
        ClustersetIP CIDR:     243.0.16.0/20

 ✓ Showing versions
COMPONENT                       REPOSITORY           CONFIGURED   RUNNING                     ARCH
submariner-gateway              quay.io/submariner   0.19.1       release-0.19-80a2c6793e9f   arm64
submariner-routeagent           quay.io/submariner   0.19.1       release-0.19-80a2c6793e9f   arm64
submariner-metrics-proxy        quay.io/submariner   0.19.1       release-0.19-35f346829412   arm64
submariner-operator             quay.io/submariner   0.19.1       release-0.19-077b8d39d39e   arm64
submariner-lighthouse-agent     quay.io/submariner   0.19.1       release-0.19-06e3b5d4a989   arm64
submariner-lighthouse-coredns   quay.io/submariner   0.19.1       release-0.19-06e3b5d4a989   arm64


Cluster "kind-backup"
 ✓ Detecting broker(s)
NAMESPACE               NAME                COMPONENTS                        GLOBALNET   GLOBALNET CIDR   DEFAULT GLOBALNET SIZE   DEFAULT DOMAINS
submariner-k8s-broker   submariner-broker   service-discovery, connectivity   no          242.0.0.0/8      65536

 ✓ Showing Connections
GATEWAY          CLUSTER        REMOTE IP    NAT   CABLE DRIVER   SUBNETS                        STATUS      RTT avg.
primary-worker   kind-primary   10.89.1.71   no    vxlan          10.110.0.0/16, 10.240.0.0/16   connected

 ✓ Showing Endpoints
CLUSTER        ENDPOINT IP   PUBLIC IP     CABLE DRIVER   TYPE
kind-backup    10.89.1.75    93.66.84.49   vxlan          local
kind-primary   10.89.1.71    93.66.84.49   vxlan          remote

 ✓ Showing Gateways
NODE            HA STATUS   SUMMARY
backup-worker   active      All connections (1) are established

 ✓ Showing Network details
    Discovered network details via Submariner:
        Network plugin:  calico
        Service CIDRs:   [10.111.0.0/16]
        Cluster CIDRs:   [10.241.0.0/16]
        ClustersetIP CIDR:     243.0.0.0/20

 ✓ Showing versions
COMPONENT                       REPOSITORY           CONFIGURED   RUNNING                     ARCH
submariner-gateway              quay.io/submariner   0.19.1       release-0.19-80a2c6793e9f   arm64
submariner-routeagent           quay.io/submariner   0.19.1       release-0.19-80a2c6793e9f   arm64
submariner-metrics-proxy        quay.io/submariner   0.19.1       release-0.19-35f346829412   arm64
submariner-operator             quay.io/submariner   0.19.1       release-0.19-077b8d39d39e   arm64
submariner-lighthouse-agent     quay.io/submariner   0.19.1       release-0.19-06e3b5d4a989   arm64
submariner-lighthouse-coredns   quay.io/submariner   0.19.1       release-0.19-06e3b5d4a989   arm64
```

### Test submariner service exposition between clusters

```bash
kubectl --context kind-primary create namespace test
kubectl --context kind-backup create namespace test
kubectl --context kind-backup -n test create deployment nginx --image=nginx
kubectl --context kind-backup -n test expose deployment nginx --port=80
subctl export service --context kind-backup --namespace test nginx

```

## Install Strimzi operator

Before deploying the Strimzi cluster operator, create a namespace called kafka:
```bash
kubectl create namespace kafka
```

Apply the Strimzi install files, including ClusterRoles, ClusterRoleBindings and some Custom Resource Definitions (CRDs):
```bash
kubectl create -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka
```

Follow the deployment of the Strimzi cluster operator:
```bash
kubectl get pod -n kafka --watch -o wide
```

## Deploy Kafka clusters

Create the primary kafka cluster:
```bash
INGRESS_NGINX_IP=$(kubectl get services \
   --namespace ingress-nginx \
   ingress-nginx-controller \
   --output jsonpath='{.spec.clusterIP}') \
envsubst < primary/kafka.yaml | kubectl apply -n kafka -f -
```

Waiting for primary cluster to be ready:
```bash
kubectl wait kafka/streams-cluster --for=condition=Ready --timeout=300s -n kafka
```

# Create a dedicated mirroring user

Create a mirroring user into the primary cluster:
```bash
kubectl apply -f primary/mirroring-user.yaml -n kafka
```

Waiting for user to be ready:
```bash
kubectl wait kafkauser/mirroring-user --for=condition=Ready --timeout=300s -n kafka
```

# Create a topic into the primary cluster 

Create a topic into the primary cluster:
```bash
kubectl apply -f primary/topic.yaml -n kafka
```

Waiting for topic to be ready:
```bash
kubectl wait kafkatopic/my-topic --for=condition=Ready --timeout=300s -n kafka
```


## Test
Start a producer attached to plain listener 
```bash
kubectl -n kafka run kafka-producer -ti --image=quay.io/strimzi/kafka:0.45.0-kafka-3.9.0 --rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server streams-cluster-kafka-bootstrap:9092 --topic my-topic
```

Get the truststore file for TLS connection to kafka cluster
```bash
kubectl get secret streams-cluster-cluster-ca-cert -n kafka -o jsonpath='{.data.ca\.crt}' | base64 -d > test/ca.crt
keytool -import -trustcacerts -alias root -file test/ca.crt -keystore test/truststore.jks -storepass password -noprompt
```

Start an external consumer attached to external listener in TLS
```bash
TEST_DIR=$(pwd)/test
bin/kafka-console-producer.sh --broker-list localhost:8443 --producer-property security.protocol=SSL --producer-property ssl.truststore.password=password --producer-property ssl.truststore.location=${TEST_DIR}/truststore.jks --topic my-topic
```

