---
layout: post
title: Streaming data to ElasticSeach using Amazon Managed Services for Kafka
summary: In this post we look at building a solution that sources data from Kafka and persists it to ElasticSeach so we can visualise using Grafana
tags: [aws]
---

### Logs of logs..with logs on the side

I was recently tasked with looking at how we could get data from on prem to a cloud based ElasticSearch cluster using Kafka. The producer on prem would be changed to push messages to a topic and we had to come up with options to get these into ElasticSearch where we would use Grafana to dashboard.

The initial proof of concept was to get a solution running 
* Producer - Lambda running within the VPC sending data to MSK
* MSK Cluster - Kafka cluster
* Consumer -  Lambda running within the VPC, consuming MSK topic and sending data  to Elastic
* ElasticSearch - ElasticSearch cluster
* Grafana - container running in EKS to display dashboards from ElasticSearch
* Aurora - DB for running the Grafana DB

### Prerequisites
You will need the following installed
* git
* AWS CLI
* AWS CDK
* docker
* jq

ElasticSearch requires a ServiceLinked role, this is only needed for new accounts but as it is not stack specific CDK/CFN deployment is not recommend so needs to be run using the following CLI command:
```
aws iam create-service-linked-role --aws-service-name es.amazonaws.com
```

### Deploy the infrastructure using CDK
```bash
#Clone the repo
git clone https://github.com/msimpsonnz/aws-misc
#Setup some env variables
export CDK_AWS_REGION=ap-southeast-2
export CDK_AWS_ACCOUNT=$(aws sts get-caller-identity | jq -r .Account)
#Build Infra with CDK
cd msk-logs/cdk
cdk deploy
```

### Setup the MSK environment
```bash 
#Setup up our MSK environment
cd ..
export AWS_KAFKA_TOPIC=AWSKafkaTutorialTopic
export AWS_MSK_CLUSTER=$(aws cloudformation describe-stack-resources --stack-name msk-demo-stack | jq -r '.StackResources[] | select(.ResourceType == "AWS::MSK::Cluster") | .PhysicalResourceId')
echo $AWS_MSK_CLUSTER
export AWS_MSK_CLUSTER_CONNECTSTRING=$(aws kafka describe-cluster --region $CDK_AWS_REGION --cluster-arn $AWS_MSK_CLUSTER | jq -r ".ClusterInfo.ZookeeperConnectString")
#If you get an error here make sure the cluster is "ACTIVE"
echo $AWS_MSK_CLUSTER_CONNECTSTRING

export AWS_MSK_BOOTSTRAP=$(aws kafka get-bootstrap-brokers --region $CDK_AWS_REGION --cluster-arn $AWS_MSK_CLUSTER | jq -r .BootstrapBrokerString)
echo $AWS_MSK_BOOTSTRAP
```

Now you will need to update the Lambda Functions environment variables with the `AWS_MSK_BOOTSTRAP` setting as it takes a while for the cluster to come online and these settings to be made available.

You also need to create a topic on Kafka, you can boot an EC2 instance for this or deploy a container to EKS to get the job done.

### Build Java Container
```bash
#Docker if required
#Build the base image
$(aws ecr get-login --no-include-email --region $CDK_AWS_REGION)
cd ../src/kafka-base
curl https://archive.apache.org/dist/kafka/2.2.1/kafka_2.12-2.2.1.tgz -o ./bin/kafka_2.12-2.2.1.tgz
docker build -t kafka-base .
aws ecr create-repository --repository-name kafka-base

docker tag kafka-base:latest $CDK_AWS_ACCOUNT.dkr.ecr.$CDK_AWS_REGION.amazonaws.com/kafka-base:latest
docker push $CDK_AWS_ACCOUNT.dkr.ecr.$CDK_AWS_REGION.amazonaws.com/kafka-base:latest
```

Deploy this to EKS and then access via the shell and run the command below, you can get the connection string from the previous command section.
```
bin/kafka-topics.sh --create --zookeeper ZookeeperConnectString --replication-factor 3 --partitions 1 --topic AWSKafkaTutorialTopic
```

Now on to the final step of configuring Grafana, its a simple deployment using Helm and then once logged on you can update the config to use mySQL with Aurora and connect the dashboards to ElasticSearch.

### Grafana setup
```bash
aws eks --region $CDK_AWS_REGION update-kubeconfig --name msk-EKSCluster --profile default

export CDK_AWS_REGION=ap-southeast-2
export CDK_AWS_ACCOUNT=$(aws sts get-caller-identity | jq -r .Account)
export ASSUME_ROLE=$(aws sts assume-role --role-arn arn:aws:iam::$CDK_AWS_ACCOUNT:role/msk-demo-stack-AdminRole38563C57-1PGU4XTWJAHY6 --role-session-name AWSCLI-Session)

export AWS_ACCESS_KEY_ID=$(echo $ASSUME_ROLE | jq -r .Credentials.AccessKeyId)
export AWS_SESSION_TOKEN=$(echo $ASSUME_ROLE | jq -r .Credentials.SessionToken)
export AWS_SECRET_ACCESS_KEY=$(echo $ASSUME_ROLE | jq -r .Credentials.SecretAccessKey)
aws sts get-caller-identity

kubectl apply -f ./src/eks/rbac.yaml

kubectl create namespace grafana
helm install stable/grafana \
    --name gf-release \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set adminPassword="EKSsAWSome" \
    --set service.type=LoadBalancer
```

That's it! You can now connect Grafana to ElasticSearch and have the data flow from Lambda into MSK and back out.