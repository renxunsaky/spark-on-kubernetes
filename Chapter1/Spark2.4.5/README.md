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
xunren@Xuns-MBP ~/workspace/spark $ curl -s -k https://www-eu.apache.org/dist/spark/spark-2.4.5/spark-2.4.5-bin-hadoop2.7.tgz -o spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP ~/workspace/spark $ ll
total 327920
-rw-r--r--  1 xunren  staff   160M Apr 28 23:47 spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP ~/workspace/spark tar xvfz spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP  ~/workspace/spark  rm -f spark-2.4.5-bin-hadoop2.7.tgz
xunren@Xuns-MBP  ~/workspace/spark  cd spark-2.4.5-bin-hadoop2.7
```
* Build Image

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
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
renxunsaky/spark-r                    latest              27000351ba0c        4 minutes ago       1.03GB
renxunsaky/spark-py                   latest              a0008748d654        6 minutes ago       983MB
renxunsaky/spark                      latest              0458196ef28f        7 minutes ago       482MB
```

## Step 2: Launch Spark Pi in cluster mode
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

Submit the job(here we **don't need** to specify the schema of jar file as local:///path_to_jar as documentation described):
```
bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.container.image.pullPolicy=Never
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
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode client \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.container.image.pullPolicy=Never
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
If we get the error : ErrImageNeverPull, it is because we're setting the conf to never pull image from remote, 
however, it cannot find the corresponding image from local docker images. In that case, don't forget to :
1. Verify if you have done the command: eval $(minikube docker-env)
2. If you did the first verification, make sure you have the image by doing the command : docker images
 (this command should be done in the same terminal session of the command "eval $(minikube docker-env)")
3. If the image is not present in the minikube context, by do exist in the docker context
(open a new terminal session without doing the command "eval $(minikube docker-env)"), then we could use these commands:
    
    3.1 minikube cache add alpine:latest => push image to minikube. The add command will store the requested image to $MINIKUBE_HOME/cache/images
    3.2 minikube cache reload => To reload all the cached images on demand
    3.3 minikube cache list => To display images you have added to the cache:
    3.4 minikube cache delete <image name> => To delete cached image
 

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

If we get the following error:
```
Caused by: io.fabric8.kubernetes.client.KubernetesClientException: Failure executing: GET at: 
https://kubernetes.default.svc/api/v1/namespaces/default/pods/spark-pi-aae2f172e85ed3b7-driver. 
Message: Forbidden!Configured service account doesn't have access. Service account may have been revoked.
 pods "spark-pi-aae2f172e85ed3b7-driver" is forbidden: User "system:serviceaccount:default:default" cannot get 
resource "pods" in API group "" in the namespace "default".
```
we need to type this command:
```
kubectl create clusterrolebinding default-admin --clusterrole cluster-admin --serviceaccount=default:default
```