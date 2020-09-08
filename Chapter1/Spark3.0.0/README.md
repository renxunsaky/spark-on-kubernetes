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
 ./bin/docker-image-tool.sh -r spark3 -p kubernetes/dockerfiles/spark/bindings/python/Dockerfile build

  xunren@Xuns-MBP $ ~/workspace/spark/spark-3.0.0-bin-hadoop3.2 $ docker images
 REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
 spark3/spark-py                      latest              e84ea127027d        29 seconds ago      1.08GB
 spark3/spark                         latest              06c7c9252a91        7 minutes ago       609MB
```

## Step 2: Launch Spark Pi in cluster mode
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

Submit the job (here we have to specify the schema of jar file as local:///path_to_jar):
```
 bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode cluster \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.container.image.pullPolicy=Never
    --conf spark.executor.instances=2 \
    --conf spark.kubernetes.container.image=spark3/spark \
    --conf spark.executor.memory=512m --conf spark.driver.memory=512m \
    local:///opt/spark/examples/jars/spark-examples_2.12-3.0.0.jar
```

**Verification:**
When the command is launched, we could check the pods with the following command
```
 xunren@Xuns-MBP $ ~ $ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
spark-pi-9c7ec571d23c8a26-driver   1/1     Running   0          18s
 xunren@Xuns-MBP $ ~ $ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
spark-pi-9c7ec571d23c8a26-driver   1/1     Running   0          38s
spark-pi-9c7ec571d23c8a26-exec-1   1/1     Running   0          19s
spark-pi-9c7ec571d23c8a26-exec-2   1/1     Running   0          19s
```
we could also check the logs of the containers. With the option "-f", it will show the logs in stream mode
```
  xunren@Xuns-MBP $ ~ $ kubectl log spark-pi-9c7ec571d23c8a26-driver
 log is DEPRECATED and will be removed in a future version. Use logs instead.
 ++ id -u
 + myuid=185
 ++ id -g
 + mygid=0
 + set +e
 ++ getent passwd 185
 + uidentry=
 + set -e
...
Pi is roughly 3.1472557362786815
...
```

**Cleanup the driver container in order to release resources**
```
 xunren@Xuns-MBP $ ~ $ kubectl get pods
 NAME                               READY   STATUS      RESTARTS   AGE
 spark-pi-9c7ec571d23c8a26-driver   0/1     Completed   0          12m
  xunren@Xuns-MBP $ ~ $ kubectl delete pod spark-pi-9c7ec571d23c8a26-driver
 pod "spark-pi-9c7ec571d23c8a26-driver" deleted
```

## Step 3: Launch Spark Pi in client mode
In the client mode, the driver will be launched from where the spark-submit command is triggered. So we need to refer to
the main jar by local path here.
```
 bin/spark-submit \
    --master k8s://https://192.168.64.2:8443 \
    --deploy-mode client \
    --name spark-pi \
    --class org.apache.spark.examples.SparkPi \
    --conf spark.kubernetes.container.image.pullPolicy=Never
    --conf spark.executor.instances=3 \
    --conf spark.kubernetes.container.image=spark3/spark \
    --conf spark.executor.memory=512m \
    /Users/xunren/workspace/spark/spark-3.0.0-bin-hadoop3.2/examples/jars/spark-examples_2.12-3.0.0.jar
```

Since the driver is not launched in the Kubernetes cluster, we can see only the pods of executors
```
 xunren@Xuns-MBP $ ~ $ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
spark-pi-fd411271d24c46f2-exec-1   1/1     Running   0          6s
spark-pi-fd411271d24c46f2-exec-2   1/1     Running   0          6s
spark-pi-fd411271d24c46f2-exec-3   1/1     Running   0          6s
```

## Debugging
This part should be the same as [Spark 2.4.5](../Spark2.4.5/README.md#Debugging)