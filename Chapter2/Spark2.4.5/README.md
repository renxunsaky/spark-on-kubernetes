# Simple Spark on Kubernetes on local PC with some advanced settings
Based on Chapter1, we will add some advanced configuration, such as
 * volume
 * namespace
 * service account
 * secret
 * Imperative object configuration

## Prerequisite:
* Docker Desktop installed (I am using 2.2.0.5 Community version)
* Minikube installed
* Docker configured with at least 6GB memory and 4 CPUs
* Since we're going to use Spark History Server, we need to modify the original Dockerfile
* Confirm the following commands work fine
```
xunren@Xuns-MBP ~ kubectl cluster-info
Kubernetes master is running at https://kubernetes.docker.internal:6443
KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```


```
xunren@Xuns-MBP ~ docker info
Client:
 Debug Mode: false

Server:
 Containers: 25
  Running: 20
  Paused: 0
  Stopped: 5
 Images: 40
 Server Version: 19.03.8
 Storage Driver: overlay2
  Backing Filesystem: <unknown>
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Plugins:
 ....
```

## Step 1: Build Docker image

The first thing to do is building the Docker image which will be used by K8S to instantiate the pods
In this step, we're going to modify the original provided Dockerfile by installing dependency "procps" in addition
It is used in the script "start-history-server.sh" which requires the command "ps"

`Note:  In any case, if you want to debug the docker image, you could override the entrypoint.sh by doing:
sudo docker run --entrypoint [new_command] [docker_image] [optional:value]
ex. sudo docker run -it --entrypoint /bin/bash [docker_image]`

* Download Apache Spark
```
xunren@Xuns-MBP ~/workspace/spark $ curl -s -k https://www-eu.apache.org/dist/spark/spark-2.4.5/spark-2.4.5-bin-hadoop2.7.tgz -o spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP ~/workspace/spark $ ll
total 327920
-rw-r--r--  1 xunren  staff   160M Apr 28 23:47 spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP ~/workspace/spark tar xvfz spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP  ~/workspace/spark  rm -f spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP  ~/workspace/spark  cd spark-2.4.5-bin-hadoop2.7
```
* Build Image

As said above, we need to modify the Dockerfile:
Add "procps" in the line:
```
apt install -y bash procps tini libc6 libpam-modules libnss3 && \
```

Besides we will add a line to copy the customised conf file for Spark
This file should be put into the downloaded directory "conf" of Spark
```
COPY conf/spark-defaults.conf /opt/spark/conf/
```
You could find the example file in the "conf" directory of this project

Spark ships with a bin/docker-image-tool.sh script that can be used to build and publish the Docker images to use with the Kubernetes backend.  
```
./bin/docker-image-tool.sh -r renxunsaky build
```
As we're working on our Local PC, we don't need to push the image onto our Docker registry. This script will use the Dockerfile available in the directory "./kubernetes/dockerfiles/spark"
If we look into the content of this file, we will find that it will copy the contents of the directories ./bin, ./sbin, ./kubernetes/dockerfiles/spark/entrypoint.sh, ./data etc. into /opt/.. 
of the target image. The entry point will be /opt/entrypoint.sh. So if you encountered some errors or you want to know what it does while starting the pod, you could look into this file. 
```
COPY ${spark_jars} /opt/spark/jars
COPY bin /opt/spark/bin
COPY sbin /opt/spark/sbin
COPY conf/spark-defaults.conf /opt/spark/conf/
COPY ${img_path}/spark/entrypoint.sh /opt/
COPY examples /opt/spark/examples
COPY ${k8s_tests} /opt/spark/tests
COPY data /opt/spark/data

ENV SPARK_HOME /opt/spark

WORKDIR /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]
```
* Check the images
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ docker images
REPOSITORY                           TAG                    IMAGE ID            CREATED             SIZE
renxunsaky/spark-r                   latest                 5b8d1f01054b        2 minutes ago       1.11GB
renxunsaky/spark-py                  latest                 101c8a2a1da2        4 minutes ago       1.06GB
renxunsaky/spark                     latest                 99b50e3f4c5e        6 minutes ago       556MB
```

## Setup Kubernetes environment
Create namespace named "trading"
```
 xunren@Xuns-MBP  ~  kubectl create namespace trading
namespace/trading created
```
Create service account named "adm-devtrade01"
```
 xunren@Xuns-MBP  ~  kubectl create serviceaccount adm-devtrade01 -n trading
serviceaccount/adm-devtrade01 created
```

Create role binding between the service account create above and the role "admin".
There are two levels of role binding
* ClusterRoleBinding: binds a cluster role to a service account.
A ClusterRole can be used to grant access to cluster-scoped resources (like nodes) 
as well as namespaced resources (like pods) across all namespaces
* RoleBinding: binds a role to a service account. 
A Role can only be used to grant access to resources (like pods) within a single namespace

For Spark on Kubernetes, since the driver always creates executor pods in the same namespace, a Role is sufficien

```
 xunren@Xuns-MBP  ~  kubectl create rolebinding spark-devtrade01 --clusterrole=admin --serviceaccount=trading:adm-devtrade01 -n trading
rolebinding.rbac.authorization.k8s.io/spark-devtrade01 created

 xunren@Xuns-MBP  ~  kubectl get rolebindings -n trading
NAME               AGE
spark-devtrade01   73s
```
For the role binding creation, in fact, for the option "serviceaccount", we need to declare the value like 
$namespace:$serviceaccount. You could remove the last option "-n trading".

## Step 2: Create Persistent Volume and its claims
In order to test it on local, we will use "hostPath" for persistence volume
The path "/mnt/data/history/spark" should exist on the K8S node:
```
  hostPath:
    path: "/mnt/data/history/spark"
```
We will use "minikube ssh" to connect to the node and create the directory
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-2.4.5-bin-hadoop2.7  minikube ssh
docker@minikube:~$ mkdir -p /mnt/data/history/spark
```

After the host path on the node is created, we can go ahead to create the persistence volume and its claim
```
kubectl apply -f storage.yml
```

## Step 3: Create Spark history pod and service
In order to use the local build docker images, we need to :
 - export some environment variables by doing this command:
 ```
    eval $(minikube docker-env)
 ```

 - add the following line in the pod specification (see the file [spark-history.yml](conf/spark-history.yml))
 ```
 command: [ "/opt/spark/sbin/start-history-server.sh" ]
 ```

In order to let the container for Spark History Server continues to be up, it is very important to set the
environment variable "SPARK_NO_DAEMONIZE"
I put it into the [Dockerfile](build_files/Dockerfile):
```
ENV SPARK_NO_DAEMONIZE TRUE
```

Finally, we can create the pod and the service
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf  kubectl apply -f spark-history.yml
pod/history-mt4 created
service/history-mt4 unchanged
 xunren@Xuns-MBP  ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf  kubectl get pods -n trading
NAME          READY   STATUS    RESTARTS   AGE
history-mt4   1/1     Running   0          3s
```

To connect to the Spark History UI on the PC:
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf  kubectl get all -n trading
NAME              READY   STATUS    RESTARTS   AGE
pod/history-mt4   1/1     Running   0          21m

NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/history-mt4   NodePort   10.104.58.194   <none>        8888:30287/TCP   23h
 xunren@Xuns-MBP  ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf  minikube service history-mt4 --url -n trading
🏃  Starting tunnel for service history-mt4.
|-----------|-------------|-------------|------------------------|
| NAMESPACE |    NAME     | TARGET PORT |          URL           |
|-----------|-------------|-------------|------------------------|
| trading   | history-mt4 |             | http://127.0.0.1:53214 |
|-----------|-------------|-------------|------------------------|
http://127.0.0.1:53214
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

## Step 4: Launch Spark Pi in cluster mode
In cluster mode, the driver will be launched on one of nodes of the cluster. And we have seen 
above (in the file /opt/entrypoint.sh), the files are copied into /opt/spark/..., so the path of the jar
should be /opt/spark/...

In order to determine the master of Kubernetes, we can use the command:
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl cluster-info
Kubernetes master is running at https://kubernetes.docker.internal:6443
KubeDNS is running at https://kubernetes.docker.internal:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

Submit the job(here we **don't need** to specify the schema of jar file as local:///path_to_jar as documentation described):
```
bin/spark-submit \
    --master k8s://https://kubernetes.docker.internal:6443 \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=renxunsaky/spark \
    --conf spark.executor.memory=512m --conf spark.driver.memory=512m \
    /opt/spark/examples/jars/spark-examples_2.11-2.4.5.jar
```

**Verification:**
When the command is launched, we could check the pods with the following command
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get pods
NAME                            READY   STATUS              RESTARTS   AGE
spark-pi-1588280749633-driver   0/1     ContainerCreating   0          2s
```
we could also check the logs of the containers. With the option "-f", it will show the logs in stream mode
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl log -f spark-pi-1588280749633-driver
log is DEPRECATED and will be removed in a future version. Use logs instead.
++ id -u
+ myuid=0
++ id -g
+ mygid=0
+ set +e
++ getent passwd 0
+ uidentry=root:x:0:0:root:/root:/bin/bash
...
```

**Cleanup the driver container in order to release resources**
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get pods
NAME                            READY   STATUS      RESTARTS   AGE
spark-pi-1588280749633-driver   0/1     Completed   0          9m47s
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl delete pod spark-pi-1588280749633-driver
pod "spark-pi-1588280749633-driver" deleted
```

## Step 3: Launch Spark Pi in client mode
In the client mode, the driver will be launched from where the spark-submit command is triggered. So we need to refer to
the main jar by local path here.
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ bin/spark-submit \
    --master k8s://https://kubernetes.docker.internal:6443 \
    --deploy-mode client \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.executor.instances=3 \
    --conf spark.kubernetes.container.image=renxunsaky/spark \
    --conf spark.executor.memory=512m \
    /Users/xunren/workspace/spark/spark-2.4.5-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.5.jar
```

Since the driver is not launched in the Kubernetes cluster, we can see only the pods of executors
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get pods
NAME                            READY   STATUS      RESTARTS   AGE
spark-pi-1588281256143-exec-1   1/1     Running     0          14s
spark-pi-1588281257099-exec-2   1/1     Running     0          13s
spark-pi-1588281257172-exec-3   1/1     Running     0          13s
```

## Debugging
If there is any error or if the driver is not launched, we can use kubectl to get more hints
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get node
NAME             STATUS   ROLES    AGE   VERSION
docker-desktop   Ready    master   12d   v1.15.5

 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl describe node
Name:               docker-desktop
Roles:              master
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=docker-desktop
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/master=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sat, 18 Apr 2020 22:32:09 +0200
Taints:             <none>
Unschedulable:      false
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Thu, 30 Apr 2020 23:17:22 +0200   Wed, 29 Apr 2020 00:50:48 +0200   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Thu, 30 Apr 2020 23:17:22 +0200   Wed, 29 Apr 2020 00:50:48 +0200   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Thu, 30 Apr 2020 23:17:22 +0200   Wed, 29 Apr 2020 00:50:48 +0200   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Thu, 30 Apr 2020 23:17:22 +0200   Wed, 29 Apr 2020 00:50:48 +0200   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  192.168.65.3
  Hostname:    docker-desktop
Capacity:
 cpu:                4
 ephemeral-storage:  61255492Ki
 hugepages-1Gi:      0
 ...
 ...
 ```
 
 Describe pods:
 ```
  xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl describe pod spark-pi-1588281590000-driver
 Name:         spark-pi-1588281590000-driver
 Namespace:    default
 Priority:     0
 Node:         docker-desktop/192.168.65.3
 Start Time:   Thu, 30 Apr 2020 23:19:51 +0200
 Labels:       spark-app-selector=spark-563e6857043d4bd089c55d45d97d8c0c
               spark-role=driver
 Annotations:  <none>
 Status:       Succeeded
 IP:           10.1.0.138
 ...
 ...
 Events:
   Type     Reason       Age   From                     Message
   ----     ------       ----  ----                     -------
   Normal   Scheduled    25s   default-scheduler        Successfully assigned default/spark-pi-1588281590000-driver to docker-desktop
   Warning  FailedMount  25s   kubelet, docker-desktop  MountVolume.SetUp failed for volume "spark-conf-volume" : configmap "spark-pi-1588281590000-driver-conf-map" not found
   Normal   Pulled       23s   kubelet, docker-desktop  Container image "renxunsaky/spark" already present on machine
   Normal   Created      23s   kubelet, docker-desktop  Created container spark-kubernetes-driver
   Normal   Started      23s   kubelet, docker-desktop  Started container spark-kubernetes-driver
```