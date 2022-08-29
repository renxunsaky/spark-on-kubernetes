# AWS EMR on AWS EKS with some advanced settings
We will use [EMR on EKS](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/emr-eks.html) provided by AWS

## Prerequisite:
* Docker Desktop installed (I am using 4.11.1 Community version)
* [kubectl](https://kubernetes.io/docs/tasks/tools/) installed
* [aws-cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed
* EKS installed: please check my [another repository](https://github.com/renxunsaky/aws-eks-infra) to deploy AWS EKS with [Terraform](http://terraform.io/) / [Terragrunt](https://terragrunt.gruntwork.io)
* Confirm the following commands work fine
```
 xunren@MBP-de-Xun  ~/workspace/personal/spark-on-kubernetes   master  kubectl cluster-info
Kubernetes control plane is running at https://xxxxxx.gr7.eu-west-1.eks.amazonaws.com
CoreDNS is running at https://xxxxxx.gr7.eu-west-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Important points to take into consideration before starting
* Use [Karpenter](https://karpenter.sh/) or other efficient Kubernetes auto-scaling tools

  For a production-ready cluster, it's very important to let the cluster auto-scale to run different sized workloads. Karpenter is known for its efficiency
  and accuracy. You could check out more its advantages on its website. With Karpenter, it's important to define good provisioners. In my example, I am using
  two different provisioners for [Spark Driver](https://github.com/renxunsaky/aws-eks-infra/blob/develop/terraform/aws-eks-karpenter/main.tf#L59-L108)
  and [Spark Executor](https://github.com/renxunsaky/aws-eks-infra/blob/develop/terraform/aws-eks-karpenter/main.tf#L110-L159).
  * I have added a label "sparkRole" for each. And when I submit a job to the cluster, I am using a [node selector](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector) to differentiate them.
  * I'm using on-demand instances for drivers, and spot instances for executors. On-demand instances for keeping the Spark application not been killed by AWS. And Spot instances for saving money.
  * It's important to choose the correct instance-family and instance-size for your drivers and executors. You should also consider to well configure your Spark applications' memory/vcore configuration in order to
    fit into the EKS nodes' size exactly.
  * It's good to put different size of VMs of the same family into a provisioner, but I don't recommend to put a lot of different instance families into the same provisioner unless you add a node selector or affinity when you
    request resources. Because, each family has a different ratio of memory/vCPU. You may waste resources if it is not well sized.
* In order to work with a encrypted S3 bucket, we need to give the job execution role corresponding KMS permission as well as a spark configuration while submitting the job
  * spark.hadoop.fs.s3a.aws.credentials.provider=com.amazonaws.auth.DefaultAWSCredentialsProviderChain

## Step 1: Build Docker image based on [AWS provided one(s)](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/docker-custom-images-steps.html)

The first thing to do is building the Docker image which will be used by K8S to instantiate the pods

* Build Image 
```
FROM 483788554619.dkr.ecr.eu-west-1.amazonaws.com/spark/emr-6.7.0:latest
USER root

RUN pip3 install --upgrade boto pyspark==3.2.1 CurrencyConverter==0.17

USER hadoop:hadoop
```
* Push the image to ECR (please create the ECR repository first if it doesn't exist)
```
 xunren@MBP-de-Xun  ~/workspace/eks/emr_on_eks  export DOCKER_ACC=123456789.dkr.ecr.eu-west-1.amazonaws.com
 xunren@MBP-de-Xun  ~/workspace/eks/emr_on_eks  export DOCKER_REPO=data/emr
 xunren@MBP-de-Xun  ~/workspace/eks/emr_on_eks  export IMG_TAG=1.1
 xunren@MBP-de-Xun  ~/workspace/eks/emr_on_eks  docker build -t $DOCKER_ACC/$DOCKER_REPO:$IMG_TAG .
[+] Building 0.3s (6/6) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                                                                                                                                            0.1s
 => => transferring dockerfile: 37B                                                                                                                                                                                                                                                             0.0s
 => [internal] load .dockerignore                                                                                                                                                                                                                                                               0.0s
 => => transferring context: 2B                                                                                                                                                                                                                                                                 0.0s
 => [internal] load metadata for 123456789.dkr.ecr.eu-west-1.amazonaws.com/spark/emr-6.7.0:latest                                                                                                                                                                                            0.0s
 => [1/2] FROM 483788554619.dkr.ecr.eu-west-1.amazonaws.com/spark/emr-6.7.0:latest                                                                                                                                                                                                              0.0s
 => CACHED [2/2] RUN pip3 install --upgrade boto pyspark==3.2.1 CurrencyConverter==0.17                                                                                                                                                                                                         0.0s
 => exporting to image                                                                                                                                                                                                                                                                          0.0s
 => => exporting layers                                                                                                                                                                                                                                                                         0.0s
 => => writing image sha256:ee833f99f0c0aa83b347e0e3423f682fa73f8fe8250e432e6198a7933b52da86                                                                                                                                                                                                    0.0s
 => => naming to 123456789.dkr.ecr.eu-west-1.amazonaws.com/datalake/emr:1.1                                                                                                                                                                                                                  0.0s

Use 'docker scan' to run Snyk tests against images to find vulnerabilities and learn how to fix them
```

## Step 2: Create a virtual EMR cluster
* Here, the EMR cluster is a virtual cluster which could be created quickly by following [this doc](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/setting-up-eks-cluster.html).
* Normally, by following the aws-eks-infra project in the prerequisite section, everything should be ready, including the necessary auto-scaling provisioner, job execution role, IAM role for service account etc. 

## Step 3: Submit the job
```
export VIRTUAL_CLUSTER_ID=$(aws emr-containers list-virtual-clusters --query "virtualClusters[?state=='RUNNING'].id" --output text)
export EMR_ROLE_ARN=$(aws iam get-role --role-name EMRContainers-JobExecutionRole --query Role.Arn --output text)
aws emr-containers start-job-run \
          --virtual-cluster-id $VIRTUAL_CLUSTER_ID \
          --name my_app \
          --execution-role-arn $EMR_ROLE_ARN \
          --release-label emr-6.7.0-latest \
          --job-driver '{
            "sparkSubmitJobDriver": {
              "entryPoint": "s3://my_bucket_name/test/eks/emails_transactions.py",
              "entryPointArguments": [
                  "20220801","my_bucket_name"
              ],
              "sparkSubmitParameters": "--conf spark.kubernetes.driver.podTemplateFile=\"'${my_bucket_name}'/test/eks/spark_driver_pod_template.yml\" --conf spark.kubernetes.executor.podTemplateFile=\"'${s3DemoBucket}'/test/eks/spark_executor_pod_template.yml\" --conf spark.executor.memory=20G --conf spark.driver.memory=8G --conf spark.executor.cores=3 --conf spark.hadoop.fs.s3a.aws.credentials.provider=com.amazonaws.auth.DefaultAWSCredentialsProviderChain --conf spark.driver.cores=4"}}' \
          --configuration-overrides '{
                "applicationConfiguration": [
                    {
                        "classification": "spark-defaults",
                        "properties": {
                          "spark.dynamicAllocation.enabled": "true",
                          "spark.shuffle.service.enabled": "true",
                          "spark.dynamicAllocation.initialExecutors": "1",
                          "sspark.dynamicAllocation.minExecutors": "1",
                          "sspark.dynamicAllocation.maxExecutors": "20",
                          "spark.kubernetes.executor.deleteOnTermination": "true",
                          "spark.sql.legacy.timeParserPolicy": "LEGACY",
                          "spark.kubernetes.container.image": "123456789.dkr.ecr.eu-west-1.amazonaws.com/data/emr:1.1"

                        }
                    }
                ],
                "monitoringConfiguration": {
                    "cloudWatchMonitoringConfiguration": {
                        "logGroupName": "/emr-on-eks/data-dev-core",
                        "logStreamNamePrefix": "my_app"
                    },
                    "s3MonitoringConfiguration": {
                        "logUri": "'${my_bucket_name}'/test/eks/logs/"
                    }
                }
    }'
```
