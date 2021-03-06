# 步骤 2 创建 EKS 集群

2.1 使用 eksctl 创建 EKS 集群(操作需要 10-15 分钟),该命令同时会创建一个使用 t3.small 的受管节点组。

详细参考手册

- [creating-and-managing-clusters](https://eksctl.io/usage/creating-and-managing-clusters/)
- [managing-nodegroups](https://eksctl.io/usage/managing-nodegroups/)

```bash
#可以修改
--node-type 工作节点类型
--nodes 工作节点数量
CLUSTER_NAME 集群名称
AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区

AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=gcr-zhy-eksworkshop

eksctl create cluster --name=${CLUSTER_NAME} --version 1.14 \
 --nodes=2 --node-type t3.medium --managed --alb-ingress-access --region=${AWS_REGION}

# update for 1.18
eksctl create cluster --name=${CLUSTER_NAME} --version 1.18 \
--region=${AWS_REGION} --nodegroup-name ${CLUSTER_NAME}-nodes \
--nodes 3 --node-type t3.medium --nodes-min 1 --nodes-max 4 --with-oidc \
--ssh-access --ssh-public-key key-pair-cn-northwest1 \
--managed
```

参考输出

```bash
[ℹ]  eksctl version 0.15.0-rc.1
[ℹ]  using region cn-northwest-1
[ℹ]  setting availability zones to [cn-northwest-1b cn-northwest-1a cn-northwest-1c]
[ℹ]  subnets for cn-northwest-1b - public:192.168.0.0/19 private:192.168.96.0/19
[ℹ]  subnets for cn-northwest-1a - public:192.168.32.0/19 private:192.168.128.0/19
[ℹ]  subnets for cn-northwest-1c - public:192.168.64.0/19 private:192.168.160.0/19
[ℹ]  using Kubernetes version 1.14
[ℹ]  creating EKS cluster "gcr-zhy-eksworkshop" in "cn-northwest-1" region with managed nodes
[ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=gcr-zhy-eksworkshop'
[ℹ]  CloudWatch logging will not be enabled for cluster "gcr-zhy-eksworkshop" in "cn-northwest-1"
[ℹ]  you can enable it with 'eksctl utils update-cluster-logging --region=cn-northwest-1 --cluster=gcr-zhy-eksworkshop'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "gcr-zhy-eksworkshop" in "cn-northwest-1"
[ℹ]  2 sequential tasks: { create cluster control plane "gcr-zhy-eksworkshop", create managed nodegroup "ng-f6ef3930" }
[ℹ]  building cluster stack "eksctl-gcr-zhy-eksworkshop-cluster"
[ℹ]  deploying stack "eksctl-gcr-zhy-eksworkshop-cluster"
[ℹ]  building managed nodegroup stack "eksctl-gcr-zhy-eksworkshop-nodegroup-ng-f6ef3930"
[ℹ]  deploying stack "eksctl-gcr-zhy-eksworkshop-nodegroup-ng-f6ef3930"
[✔]  all EKS cluster resources for "gcr-zhy-eksworkshop" have been created
[✔]  saved kubeconfig as "/Users/ruiliang/.kube/config"
[ℹ]  nodegroup "ng-f6ef3930" has 2 node(s)
[ℹ]  node "ip-192-168-14-19.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-40-132.cn-northwest-1.compute.internal" is ready
[ℹ]  waiting for at least 2 node(s) to become ready in "ng-f6ef3930"
[ℹ]  nodegroup "ng-f6ef3930" has 2 node(s)
[ℹ]  node "ip-192-168-14-19.cn-northwest-1.compute.internal" is ready
[ℹ]  node "ip-192-168-40-132.cn-northwest-1.compute.internal" is ready
[ℹ]  kubectl command should work with "/Users/ruiliang/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "gcr-zhy-eksworkshop" in "cn-northwest-1" region is ready

```

集群创建完毕后,查看 EKS 集群工作节点

```bash
 kubectl get node
```

参考输出

```bash
NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-14-19.cn-northwest-1.compute.internal    Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-40-132.cn-northwest-1.compute.internal   Ready    <none>   4d1h   v1.14.9-eks-1f0ca9

```

（可选）配置环境变量用于后续使用

```bash
STACK_NAME=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].StackName')
echo $STACK_NAME
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME --region=${AWS_REGION} | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo $ROLE_NAME
export ROLE_NAME=${ROLE_NAME}
export STACK_NAME=${STACK_NAME}
```

2.2 部署一个 nginx 测试 eks 集群基本功能

> 下载本 git repository, 参考 resource/nginx-app 目录的 nginx-nlb.yaml, 创建一个 nginx pod，并通过 LoadBalancer 类型对外暴露

```bash
git clone git@github.com:liangruibupt/EKS-Workshop-China.git
kubectl create namespace test
kubectl apply -f nginx-nlb.yaml --namespace test

## Check deployment status
kubectl get pods -n test
kubectl get deployment nginx-nlb-deployment -n test

## Get the external access 确保 EXTERNAL-IP是一个有效的AWS Network Load Balancer的地址
kubectl get service service-nginx-nlb -o wide -n test

## 下面可以试着访问这个地址
ELB=$(kubectl get service service-nginx -n test -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```

> 清理

```bash
kubectl delete -f nginx-nlb.yaml -n test
```

2.3 扩展集群节点

> 我们之前通过 eksctl 创建了一个 2 节点的集群，下面我们来扩展集群节点到 3

```bash
NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --region=${AWS_REGION}
```

> 检查结果

```bash
eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION}
kubectl get node
```

参考输出

```bash
CLUSTER			NODEGROUP	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID
gcr-zhy-eksworkshop	ng-f6ef3930	2020-03-03T08:09:39Z	2		3		3			t3.medium

NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-14-19.cn-northwest-1.compute.internal    Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-40-132.cn-northwest-1.compute.internal   Ready    <none>   4d1h   v1.14.9-eks-1f0ca9
ip-192-168-86-36.cn-northwest-1.compute.internal    Ready    <none>   3d4h   v1.14.9-eks-1f0ca9
```

2.4 （可选）中国区镜像处理

由于防火墙或安全限制，海外 gcr.io, quay.io 的镜像可能无法下载，为了不手动修改原始 yaml 文件的镜像路径，采用下面 webhook 的方式，自动修改国内配置的镜像路径。

### 快速上手版

```bash
git clone https://github.com/nwcdlabs/container-mirror.git
cd container-mirror
kubectl apply -f webhook/mutating-webhook.yaml

# 验证 pod 详细信息中的image 已经替换为本项目对应的 ECR 镜像仓库
kubectl run --generator=run-pod/v1 test --image=k8s.gcr.io/coredns:1.3.1
kubectl get pod test -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/coredns:1.3.1

# 清理
kubectl delete pod test
```

### 自己部署 Amazon API Gateway 和 Webhook

详情参考 [amazon-api-gateway-mutating-webhook-for-k8](https://github.com/aws-samples/amazon-api-gateway-mutating-webhook-for-k8)

1. 克隆 amazon-api-gateway-mutating-webhook-for-k8

````bash
```bash
git clone git@github.com:aws-samples/amazon-api-gateway-mutating-webhook-for-k8.git
cd amazon-api-gateway-mutating-webhook-for-k8
````

2. 修改 lambda_function.py 中 image_mirrors 下面的镜像地址为你偏好的国内镜像地址

   例如采用 https://github.com/nwcdlabs/container-mirror 上列出的镜像地址

```bash
# modify the lambda_function.py
image_mirrors = {
  '/': '048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/dockerhub/',
  'quay.io/': '048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/quay/',
  'gcr.io/': '048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/',
  'k8s.gcr.io/': '048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/'
}

export S3_BUCKET=my_s3_bucket
sam package -t sam-template.yaml --s3-bucket ${S3_BUCKET} --output-template-file packaged.yaml --region ${AWS_REGION}

sam deploy -t packaged.yaml --stack-name amazon-api-gateway-mutating-webhook-for-k8 --capabilities CAPABILITY_IAM --region ${AWS_REGION}
```

2. 在 AWS Cloudformation console 找到 名称为 amazon-api-gateway-mutating-webhook-for-k8 的 stack, 拷贝 Stack 输出中的 APIGatewayURL ，

3. 修改 mutating-webhook.yaml, 替换 url 为 上述 APIGatewayURL 的拷贝值，在 EKS 集群部署 webhook

```bash
kubectl apply -f mutating-webhook.yaml
```

部署样例应用，验证 webhook 工作正常

```bash
kubectl apply -f ./nginx-gcr.yaml
kubectl get pods
kubectl get pod nginx-gcr-deployment-xxxx -o=jsonpath='{.spec.containers[0].image}'
# 结果应显示为配置的gcr.io的路径，例如按照上述image_mirrors配置，结果为：
048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/gcr/google_containers/nginx
# 清理
kubectl delete -f ./nginx-gcr.yaml
```

注意：China region EKS service have two official account id, cn-northwest-1 961992271922, cn-north-1 91830976355，因此如果你需要下载 EKS 官方的镜像，需要正确使用上面的两个 id

```bash
#例如宁夏区
#aws-efs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-efs-csi-driver

#aws-ebs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-ebs-csi-driver
```

2.5 (可选) Kubernetes service different expose type

尽管每个 Pod 都有一个唯一的 IP 地址，但是如果没有 Service，这些 IP 不会暴露在群集外部。 Service 允许您的应用程序接收流量

Services 可以根据 ServiceSpec 中定义的 Type 被暴露为不同的方式:

1. ClusterIP (default) - Exposes the Service on an internal IP in the cluster. This type makes the Service only reachable from within the cluster.
2. NodePort - Exposes the Service on the same port of each selected Node in the cluster using NAT. Makes a Service accessible from outside the cluster using <NodeIP>:<NodePort>. Superset of ClusterIP.
3. LoadBalancer - Creates an external load balancer in the current cloud (if supported) and assigns a fixed, external IP to the Service. Superset of NodePort.
4. ExternalName - Exposes the Service using an arbitrary name (specified by externalName in the spec) by returning a CNAME record with the name. No proxy is used. This type requires v1.7 or higher of kube-dns.

通过一个简单实验解释不同的 ServiceSpec 类型，详情参考

[ServiceSpec type](Service-SourceIP.md)
