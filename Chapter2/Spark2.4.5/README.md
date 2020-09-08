# Simple Spark on Kubernetes on local PC with some advanced settings
Based on Chapter1, we will add some advanced configuration, such as
 * volume
 * namespace
 * service account
 * Imperative object configuration

## Prerequisite:
* Docker Desktop installed (I am using 2.2.0.5 Community version)
* Minikube installed and started with at least 4 CPUs (minikube start --cpus 4)
* Since we're going to use Spark History Server, we need to modify the original Dockerfile
* Confirm the following commands work fine
```
xunren@Xuns-MBP ~ kubectl cluster-info
Kubernetes master is running at https://192.168.64.2:8443
KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

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
In this step, we're going to modify the original provided Dockerfile for several reasons:
1. The original Dockerfile is built from openjdk:8-jre-slim with the update java8_252, we got an error : 
[java.net.SocketException: Broken pipe (Write failed)](#known-issues)
2. The image openjdk:8-jre-slim which doesn't contain some commands and libraries, such as "ps" which is used in 
the script "start-history-server.sh".

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

The customized Dockerfile could be found [here](build_files/Dockerfile)
We have added:
1. /usr/bin/tini : which is used by the entrypoint.sh file
2. a line to copy the customised conf file for Spark. This file should be put into the downloaded directory "conf" of Spark
```
COPY conf/spark-defaults.conf /opt/spark/conf/
```
You could find the [example file](build_files/spark/conf/spark-defaults.conf) in this project

Spark ships with a bin/docker-image-tool.sh script that can be used to build and publish the Docker images to use with the Kubernetes backend.  
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ ./bin/docker-image-tool.sh -r spark2 build
```
But we could build the image with a simple docker command:
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ docker build -t spark2/spark:latest -f kubernetes/dockerfiles/spark/Dockerfile .
```
As we're working on our Local PC, we don't need to push the image onto our Docker registry. But if the image is not built
in the context of minikube, we need to put it into cache, by doing:
```
eval $(minikube docker-env)
```

* Check the images
```
xunren@Xuns-MBP  ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ docker images
REPOSITORY                           TAG                    IMAGE ID            CREATED             SIZE
spark2/spark-r                   latest                 5b8d1f01054b        2 minutes ago       1.11GB
spark2/spark-py                  latest                 101c8a2a1da2        4 minutes ago       1.06GB
spark2/spark                     latest                 99b50e3f4c5e        6 minutes ago       556MB
```

## Step 2: Setup Kubernetes environment
Create namespace named "trading"
```
 xunren@Xuns-MBP $ ~ $ kubectl create namespace trading
namespace/trading created
```
Create service account named "adm-devtrade01"
```
 xunren@Xuns-MBP $ ~ $ kubectl create serviceaccount adm-devtrade01 -n trading
serviceaccount/adm-devtrade01 created
```

Create role binding between the service account create above and the role "admin".
There are two levels of role binding
* ClusterRoleBinding: binds a cluster role to a service account.
A ClusterRole can be used to grant access to cluster-scoped resources (like nodes) 
as well as namespaced resources (like pods) across all namespaces
* RoleBinding: binds a role to a service account. 
A Role can only be used to grant access to resources (like pods) within a single namespace

For Spark on Kubernetes, since the driver always creates executor pods in the same namespace, a Role is sufficient

```
 xunren@Xuns-MBP $ ~ $ kubectl create rolebinding spark-devtrade01 --clusterrole=admin --serviceaccount=trading:adm-devtrade01 -n trading
rolebinding.rbac.authorization.k8s.io/spark-devtrade01 created

 xunren@Xuns-MBP $ ~ $ kubectl get rolebindings -n trading
NAME               AGE
spark-devtrade01   73s
```
For the role binding creation, in fact, for the option "serviceaccount", we need to declare the value like 
$namespace:$serviceaccount. You could remove the last option "-n trading".

The above manual commands could also be implemented in the way of Imperative object configuration like the file [service-account.yml](conf/service-account.yml)

## Step 3: Create Persistent Volume and its claims
In order to test it on local, we will use "hostPath" for persistence volume
The path "/mnt/data/history/spark" should exist on the K8S node:
```
  hostPath:
    path: "/mnt/data/history/spark"
```
We will use "minikube ssh" to connect to the node and create the directory
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ minikube ssh
docker@minikube:~$ sudo mkdir -p /mnt/data/history/spark
```

After the host path on the node is created, we can go ahead to create the persistence volume and its claim
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl apply -f storage.yml
```

## Step 4: Create Spark history pod and service
In order to use the local build docker images, we need to :
 - export some environment variables by doing this command:
 ```
    eval $(minikube docker-env)
 ```

 - add the following line in the pod specification (see the file [spark-history.yml](conf/spark-history.yml)) 
 in order to override the entrypoint.sh file
 ```
 command: [ "/opt/spark/sbin/start-history-server.sh" ]
 ```

In order to let the container for Spark History Server continues to run, it is very important to set the
environment variable "SPARK_NO_DAEMONIZE". Otherwise, the pod will be completed quickly after the execution of shell command
I put it into the [Dockerfile](build_files/Dockerfile):
```
ENV SPARK_NO_DAEMONIZE TRUE
```

Finally, we can create the pod and the service
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl apply -f spark-history.yml
pod/history-mt4 created
service/history-mt4 unchanged
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl get pods -n trading
NAME          READY   STATUS    RESTARTS   AGE
history-mt4   1/1     Running   0          3s
```

To connect to the Spark History UI on the PC:
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl get all -n trading
NAME              READY   STATUS    RESTARTS   AGE
pod/history-mt4   1/1     Running   0          21m

NAME                  TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/history-mt4   NodePort   10.104.58.194   <none>        8888:30287/TCP   23h
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ minikube service history-mt4 --url -n trading
🏃  Starting tunnel for service history-mt4.
|-----------|-------------|-------------|------------------------|
| NAMESPACE |    NAME     | TARGET PORT |          URL           |
|-----------|-------------|-------------|------------------------|
| trading   | history-mt4 |             | http://127.0.0.1:53214 |
|-----------|-------------|-------------|------------------------|
http://127.0.0.1:53214
❗  Because you are using a Docker driver on darwin, the terminal needs to be open to run it.
```

## Step 5: Launch Spark Pi in cluster mode
In cluster mode, the driver will be launched on one of nodes of the cluster. And we have seen 
above (in the file /opt/entrypoint.sh), the files are copied into /opt/spark/..., so the path of the jar
should be /opt/spark/...

In order to determine the master of Kubernetes, we can use the command:
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl cluster-info
Kubernetes master is running at https://192.168.64.2:8443
KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

In the following command, fwe have added the configuration for namespace, service account. The image pull policy is set
to be "Never" to avoid it pull from remote registry. In order to get the spark history logs, we're using the dedicated
configuration "spark.kubernetes.driver.volumes..." to mount the persistence volume used by Spark History Server
```
bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=adm-devtrade01 \
    --conf spark.kubernetes.namespace=trading \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark2/spark \
    --conf spark.executor.memory=512m \
    --conf spark.driver.memory=512m \
    --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.history-volume-mt4.mount.path=/spark-history
    --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.history-volume-mt4.options.claimName=history-vc-mt4
    local:///opt/spark/examples/jars/spark-examples_2.11-2.4.5.jar
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
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl log -f spark-pi-1588280749633-driver -n trading
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

When there is no log at all, it is probably that the pod's entrypoint shell is not launched. In this case, we could use
the command:
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl describe pod spark-pi-1588280749633-driver -n trading
```

**Cleanup the driver container in order to release resources**
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get pods -n trading
 NAME                            READY   STATUS      RESTARTS   AGE
 history-mt4                     1/1     Running     8          3d14h
 spark-pi-1593466501409-driver   0/1     Error       0          14h
 spark-pi-1593467912822-driver   0/1     Completed   0          14h

 xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl delete pod spark-pi-1593466501409-driver -n trading
pod "spark-pi-1593466501409-driver" deleted
```

The spark-submit command could also be completed with the Imperative object configuration like the file [spark-submit-cluster.yml](conf/spark-submit-cluster.yml)

```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl apply -f spark-submit-cluster.yml
pod/spark-pi-cluster created
  xunren@Xuns-MBP $ ~/workspace/spark/spark-2.4.5-bin-hadoop2.7 $ kubectl get all -n trading
 NAME                                READY   STATUS    RESTARTS   AGE
 pod/spark-pi-1599603481121-driver   1/1     Running   0          25s
 pod/spark-pi-1599603481121-exec-1   1/1     Running   0          12s
 pod/spark-pi-1599603481121-exec-2   1/1     Running   0          12s
 pod/spark-pi-cluster                1/1     Running   0          32s
 
 NAME                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
 service/history-mt4                         NodePort    10.103.99.103   <none>        8888:30999/TCP      74d
 service/spark-pi-1599603481121-driver-svc   ClusterIP   None            <none>        4400/TCP,7079/TCP   23s
```
Here we could see that it creates an additional pod who submits the requests. It creates also a headless service automatically for us in order
to make the driver reachable from the executors.
In the yaml file, I have added "restartPolicy: Never" at the end, because without this, it will restart the driver pod after
it has finished successfully.

## Step 6: Launch Spark Pi in client mode
```
bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode client \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=adm-devtrade01 \
    --conf spark.kubernetes.namespace=trading \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark2/spark \
    --conf spark.executor.memory=512m \
    --conf spark.driver.memory=512m \
    --conf spark.eventLog.dir=/tmp \
    --conf spark.history.fs.logDirectory=/tmp \
    /Users/xunren/workspace/spark/spark-2.4.5-bin-hadoop2.7/examples/jars/spark-examples_2.11-2.4.5.jar
``` 
Note: The client mode will not work with the default configuration in  [spark-default.conf](build_files/spark/conf/spark-defaults.conf),
 because it could not find the directory "/spark-history" declared in this file. In order to make it work, 
 we have to change the properties "spark.eventLog.dir" and "spark.history.fs.logDirectory".(not recommended)

The spark-submit command could also be completed with the Imperative object configuration like the file [spark-submit-client.yml](conf/spark-submit-client.yml)
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark2.4.5/conf $ kubectl apply -f spark-submit-client.yml
service/headless-spark-pi-service created
pod/spark-pi-client-driver created
```
Please pay attention to the official doc [here](https://spark.apache.org/docs/latest/running-on-kubernetes.html#client-mode):
  - Spark executors must be able to connect to the Spark driver over a hostname and a port that is routable from the Spark executors.
  Since the driver is launched inside a pod, we need to use a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
  to make the driver be routable from the executors.
  - If you run your Spark driver in a pod, it is highly recommended to set spark.kubernetes.driver.pod.name to the name of that pod.
   When this property is set, the Spark scheduler will deploy the executor pods with an OwnerReference, which in turn will ensure that 
   once the driver pod is deleted from the cluster, all of the application’s executor pods will also be deleted.
  - It is important to create the service before the driver pod. If not, the driver pod will fail at the first try.
  - In the yaml file, I have added "restartPolicy: Never" at the end, because without this, it will restart the driver pod after
    it has finished successfully.

## All in One
You could deploy the namespace, PV, service account, service and pods in one(we will use cluster mode here) [single file](conf/all-in-one.yml). 

## Known issues
* java.net.SocketException: Broken pipe (Write failed)
https://github.com/fabric8io/kubernetes-client/issues/2168

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

Error for Spark submitting: the reason is that the entrypoint.sh can't be launched successfully. We should check the Dockerfile.
```
Events:
  Type     Reason       Age        From               Message
  ----     ------       ----       ----               -------
  Normal   Scheduled    <unknown>  default-scheduler  Successfully assigned trading/spark-pi-1593346418741-driver to minikube
  Warning  FailedMount  32s        kubelet, minikube  MountVolume.SetUp failed for volume "spark-conf-volume" : configmap "spark-pi-1593346418741-driver-conf-map" not found
  Normal   Pulled       29s        kubelet, minikube  Container image "renxunsaky/spark" already present on machine
  Normal   Created      29s        kubelet, minikube  Created container spark-kubernetes-driver
  Warning  Failed       29s        kubelet, minikube  Error: failed to start container "spark-kubernetes-driver": Error response from daemon: OCI runtime create failed: container_linux.go:349: starting container process caused "exec: \"driver\": executable file not found in $PATH": unknown
```

In Client mode, error while creating executor pods. Because the executor pod tries to connect to the driver with the driver's name.
It is necessary to configure the property: "spark.kubernetes.driver.pod.name" to be the hostname of the driver.
```
Exception in thread "main" java.lang.reflect.UndeclaredThrowableException
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1713)
	at org.apache.spark.deploy.SparkHadoopUtil.runAsSparkUser(SparkHadoopUtil.scala:64)
	at org.apache.spark.executor.CoarseGrainedExecutorBackend$.run(CoarseGrainedExecutorBackend.scala:188)
	at org.apache.spark.executor.CoarseGrainedExecutorBackend$.main(CoarseGrainedExecutorBackend.scala:285)
	at org.apache.spark.executor.CoarseGrainedExecutorBackend.main(CoarseGrainedExecutorBackend.scala)
Caused by: org.apache.spark.SparkException: Exception thrown in awaitResult:
	at org.apache.spark.util.ThreadUtils$.awaitResult(ThreadUtils.scala:226)
	at org.apache.spark.rpc.RpcTimeout.awaitResult(RpcTimeout.scala:75)
	at org.apache.spark.rpc.RpcEnv.setupEndpointRefByURI(RpcEnv.scala:101)
	at org.apache.spark.executor.CoarseGrainedExecutorBackend$$anonfun$run$1.apply$mcV$sp(CoarseGrainedExecutorBackend.scala:201)
	at org.apache.spark.deploy.SparkHadoopUtil$$anon$2.run(SparkHadoopUtil.scala:65)
	at org.apache.spark.deploy.SparkHadoopUtil$$anon$2.run(SparkHadoopUtil.scala:64)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1698)
	... 4 more
Caused by: java.io.IOException: Failed to connect to spark-pi-client:4400
	at org.apache.spark.network.client.TransportClientFactory.createClient(TransportClientFactory.java:245)
	at org.apache.spark.network.client.TransportClientFactory.createClient(TransportClientFactory.java:187)
	at org.apache.spark.rpc.netty.NettyRpcEnv.createClient(NettyRpcEnv.scala:198)
	at org.apache.spark.rpc.netty.Outbox$$anon$1.call(Outbox.scala:194)
	at org.apache.spark.rpc.netty.Outbox$$anon$1.call(Outbox.scala:190)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
	at java.lang.Thread.run(Thread.java:745)
Caused by: java.net.UnknownHostException: spark-pi-client
```
So the Solution is 
```

```