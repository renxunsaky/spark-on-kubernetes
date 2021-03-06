# Simple Spark on Kubernetes on local PC with default settings
We will work on local PC with basic and default configuration provided by Spark

## Prerequisite:
* Docker Desktop installed (I am using 2.2.0.5 Community version)
* Minikube installed and started with at least 4 CPUs (minikube start --cpus 4)
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

* Download Apache Spark
```
 xunren@Xuns-MBP ~/workspace/spark $ xunren@Xuns-MBP $ ~/workspace/spark $ curl -sk http://apache.mirrors.ovh.net/ftp.apache.org/dist/spark/spark-3.0.0/spark-3.0.0-bin-hadoop3.2.tgz
 xunren@Xuns-MBP ~/workspace/spark $ ll
 total 327920
 -rw-r--r--@  1 xunren  staff   255M May  1 23:26 spark-3.0.0-bin-hadoop3.2.tar
 xunren@Xuns-MBP $ ~/workspace/spark $ tar xvf spark-3.0.0-bin-hadoop3.2.tar
 xunren@Xuns-MBP $ ~/workspace/spark $ rm spark-3.0.0-bin-hadoop3.2.tar
 xunren@Xuns-MBP $ ~/workspace/spark $ cd spark-3.0.0-bin-hadoop3.2
```
* Build Image

Spark ships with a bin/docker-image-tool.sh script that can be used to build and publish the Docker images to use with the Kubernetes backend.  
```
 ./bin/docker-image-tool.sh -r spark3 build
```
As we're working on our Local PC, we don't need to push the image onto our Docker registry. This script will use the Dockerfile available in the directory "./kubernetes/dockerfiles/spark"
If we look into the content of this file, we will find that it will copy the contents of the directories ./bin, ./sbin, ./kubernetes/dockerfiles/spark/entrypoint.sh, ./data etc. into /opt/.. 
of the target image. The entry point will be /opt/entrypoint.sh. So if you encountered some errors or you want to know what it does while starting the pod, you could look into this file. 
```
COPY jars /opt/spark/jars
COPY bin /opt/spark/bin
COPY sbin /opt/spark/sbin
COPY kubernetes/dockerfiles/spark/entrypoint.sh /opt/
COPY examples /opt/spark/examples
COPY kubernetes/tests /opt/spark/tests
COPY data /opt/spark/data

ENV SPARK_HOME /opt/spark

WORKDIR /opt/spark/work-dir
RUN chmod g+w /opt/spark/work-dir

ENTRYPOINT [ "/opt/entrypoint.sh" ]

# Specify the User that the actual main process will run as
USER ${spark_uid}
```
There is a line in addition at last "USER ${spark_uid}" than Spark v2.4.5.
Images built from the project provided Dockerfiles contain a default USER directive with a default UID of 185. 
Users building their own images with the provided docker-image-tool.sh script can use the -u <uid> option 
to specify the desired UID.

* Check the images
```
  xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ docker images
 REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
 spark3/spark                         latest              06c7c9252a91        4 seconds ago       609MB
```
we can see that in Spark3, it builds only the image "spark". But in version 2.4.5, it builds also spark-r and spark-py.
In version 3.x, if we want to build PySpark image or SparkR image, you could indicate the full path of Dockerfile
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  ./bin/docker-image-tool.sh -r spark3 -u 1001 -f /Users/xunren/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/build_files/Dockerfile build

 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ docker images
 REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
 spark3/spark-py                      latest              e84ea127027d        29 seconds ago      1.08GB
 spark3/spark                         latest              06c7c9252a91        7 minutes ago       609MB
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
NAME               ROLE                AGE
spark-devtrade01   ClusterRole/admin   5m25s
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
 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ minikube ssh
docker@minikube:~$ sudo mkdir -p /mnt/data/history/spark
```

After the host path on the node is created, we can go ahead to create the persistence volume and its claim
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl apply -f storage.yml
persistentvolume/app-v-mt4 created
persistentvolumeclaim/app-vc-mt4 created
persistentvolume/history-v-mt4 created
persistentvolumeclaim/history-vc-mt4 created
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
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl apply -f spark-history.yml
pod/history-mt4 created
service/history-mt4 unchanged
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl get pods -n trading
NAME          READY   STATUS    RESTARTS   AGE
history-mt4   1/1     Running   0          3s
```

To connect to the Spark History UI on the PC:
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl get all -n trading
NAME              READY   STATUS    RESTARTS   AGE
pod/history-mt4   1/1     Running   0          66s

NAME                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/history-mt4   NodePort   10.101.140.205   <none>        8888:30999/TCP   66s
 xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ minikube service history-mt4 --url -n trading
http://192.168.64.2:30999
```

## Step 5: Launch Spark Pi in cluster mode
In cluster mode, the driver will be launched on one of nodes of the cluster. And we have seen 
above (in the file /opt/entrypoint.sh), the files are copied into /opt/spark/..., so the path of the jar
should be /opt/spark/...

In order to determine the master of Kubernetes, we can use the command:
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl cluster-info
Kubernetes master is running at https://192.168.64.2:8443
KubeDNS is running at https://192.168.64.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

In the following command, fwe have added the configuration for namespace, service account. The image pull policy is set
to be "Never" to avoid it pull from remote registry. In order to get the spark history logs, we're using the dedicated
configuration "spark.kubernetes.driver.volumes..." to mount the persistence volume used by Spark History Server
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=adm-devtrade01 \
    --conf spark.kubernetes.namespace=trading \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark3/spark \
    --conf spark.executor.memory=512m \
    --conf spark.driver.memory=512m \
    --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.history-volume-mt4.mount.path=/spark-history \
    --conf spark.kubernetes.driver.volumes.persistentVolumeClaim.history-volume-mt4.options.claimName=history-vc-mt4 \
    local:///opt/spark/examples/jars/spark-examples_2.12-3.0.0.jar
```

**Verification:**
When the command is launched, we could check the pods with the following command
``` 
xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl get pods -n trading
NAME                               READY   STATUS              RESTARTS   AGE
spark-pi-8fa6ec756212aa0d-driver   0/1     ContainerCreating   0          2s
```
we could also check the logs of the containers. With the option "-f", it will show the logs in stream mode
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl log -f spark-pi-8fa6ec756212aa0d-driver -n trading
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
xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl describe pod spark-pi-1588280749633-driver -n trading
```

**Cleanup the driver container in order to release resources**
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl get pods -n trading
 NAME                               READY   STATUS      RESTARTS   AGE
 history-mt4                        1/1     Running     0          40m
 spark-pi-8fa6ec756212aa0d-driver   0/1     Completed   0          6m4s

 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl delete pod spark-pi-8fa6ec756212aa0d-driver -n trading
pod "spark-pi-8fa6ec756212aa0d-driver" deleted
```

The spark-submit command could also be completed with the Imperative object configuration like the file [spark-submit-cluster.yml](conf/spark-submit-cluster.yml)

```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl apply -f spark-submit-cluster.yml
pod/spark-pi-cluster created
  xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl get all -n trading
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
 xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode client \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=adm-devtrade01 \
    --conf spark.kubernetes.namespace=trading \
    --conf spark.kubernetes.container.image.pullPolicy=Never \
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark3/spark \
    --conf spark.executor.memory=512m \
    --conf spark.driver.memory=512m \
    --conf spark.eventLog.dir=/tmp \
    --conf spark.history.fs.logDirectory=/tmp \
    /Users/xunren/workspace/spark/spark-3.0.0-bin-hadoop3.2/examples/jars/spark-examples_2.12-3.0.0.jar
``` 
Note: The client mode will not work with the default configuration in  [spark-default.conf](build_files/spark/conf/spark-defaults.conf),
 because it could not find the directory "/spark-history" declared in this file. In order to make it work, 
 we have to change the properties "spark.eventLog.dir" and "spark.history.fs.logDirectory".(not recommended)

The spark-submit command could also be completed with the Imperative object configuration like the file [spark-submit-client.yml](conf/spark-submit-client.yml)
```
xunren@Xuns-MBP $ ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf $ kubectl apply -f spark-submit-client.yml
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
**Q1:**

If there is any error or if the driver is not launched, we can use kubectl to get more hints
```
 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl get node
NAME             STATUS   ROLES    AGE   VERSION
docker-desktop   Ready    master   12d   v1.15.5

 xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl describe node
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
  xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ kubectl describe pod spark-pi-1588281590000-driver
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

**Q2:**

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

**Q3:**

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
So the Solution is : create a headless service for spark-submit in the client mode (if the spark-submit is done in a pod)


**Q4:**

Error:
```
Events:
  Type     Reason            Age        From               Message
  ----     ------            ----       ----               -------
  Warning  FailedScheduling  <unknown>  default-scheduler  running "VolumeBinding" filter plugin for pod "spark-pi-cluster": pod has unbound immediate PersistentVolumeClaims
  Warning  FailedScheduling  <unknown>  default-scheduler  running "VolumeBinding" filter plugin for pod "spark-pi-cluster": pod has unbound immediate PersistentVolumeClaims
```
Explanation: The PVs are requested by previous pods. Even the PVC is removed, the PVs are not ready to be consumed again
(should be "available" instead of "released"). We have several options here:
1. Use the recommended way: [Dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
2. kubectl patch pv $pv_name -p '{"spec":{"claimRef": null}}'
3. Delete the PVs manually and create them again

**Q5:**

Cannot delete the PVC:
```
 xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  kubectl delete pvc history-vc-mt4 -n trading
persistentvolumeclaim "history-vc-mt4" deleted
^C => we have to stop it because it lasts too much time with finish

xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  kubectl describe pvc history-vc-mt4 -n trading
Name:          history-vc-mt4
Namespace:     trading
StorageClass:  medium
Status:        Terminating (lasts 7h10m)
Volume:        history-v-mt4
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
               pv.kubernetes.io/bound-by-controller: yes
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      150Mi
Access Modes:  RWX
VolumeMode:    Filesystem
Mounted By:    spark-pi-1599776328712-driver
               spark-pi-cluster
Events:        <none>
```

The reason is : If a user deletes a PVC in active use by a Pod, the PVC is not removed immediately. 
PVC removal is postponed until the PVC is no longer actively used by any Pods. 
Also, if an admin deletes a PV that is bound to a PVC, the PV is not removed immediately. 
PV removal is postponed until the PV is no longer bound to a PVC.

The solution:
```
xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  kubectl patch pvc history-vc-mt4 -p '{"metadata":{"finalizers":null}}' -n trading
persistentvolumeclaim/history-vc-mt4 patched

xunren@Xuns-MBP  ~/workspace/spark/spark-3.0.0-bin-hadoop3.2  kubectl get pvc -n trading
No resources found in trading namespace.
```

**Q6:**

Cannot start Spark history server:
```
xunren@Xuns-MBP  ~/workspace/spark/spark-on-kubernetes/Chapter2/Spark3.0.0/conf   master ●✚  kubectl logs -n trading pod/history-mt4
mkdir: cannot create directory ‘/opt/spark/logs’: Permission denied
chown: cannot access '/opt/spark/logs': No such file or directory
starting org.apache.spark.deploy.history.HistoryServer, logging to /opt/spark/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-history-mt4.out
/opt/spark/sbin/spark-daemon.sh: line 128: /opt/spark/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-history-mt4.out: No such file or directory
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 136: ps: command not found
/opt/spark/sbin/spark-daemon.sh: line 144: ps: command not found
failed to launch: nice -n 0 /opt/spark/bin/spark-class org.apache.spark.deploy.history.HistoryServer
tail: cannot open '/opt/spark/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-history-mt4.out' for reading: No such file or directory
full log in /opt/spark/logs/spark--org.apache.spark.deploy.history.HistoryServer-1-history-mt4.out
```

The reason is:
Images built from the project provided Dockerfiles contain a default USER directive with a default UID of 185. 
This means that the resulting images will be running the Spark processes as this UID inside the container.

See: https://spark.apache.org/docs/latest/running-on-kubernetes.html#user-identity

The solution:
Modify the Dockerfile to create a dedicated group and user "spark" with uid 1001, for example.
