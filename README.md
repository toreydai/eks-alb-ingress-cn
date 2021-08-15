## 中国区Amazon EKS中使用ALB

### 1 为集群创建OIDC提供商
1.1 查看集群的OIDC提供商URL

```
aws eks describe-cluster --name <cluster_name> --query "cluster.identity.oidc.issuer" --output text
```
输出示例：

```
https://oidc.eks.cn-north-1.amazonaws.com.cn/id/EXAMPLED539D4633E53DE1B716D3041E
```
列出账户中的IAM OIDC提供商，使用上一条命令返回的值，替换EXAMPLED539D4633E53DE1B716D3041E

```
aws iam list-open-id-connect-providers | grep <EXAMPLED539D4633E53DE1B716D3041E>
```
输出示例：

```
"Arn": "arn:aws-cn:iam::111122223333:oidc-provider/oidc.eks.cn-north-1.amazonaws.com.cn/id/EXAMPLED539D4633E53DE1B716D3041E"
```
如果输出类似上述结果，则表明OIDC提供商已经创建，否则需要创建。

1.2 执行如下命令创建OIDC提供商，将<cluster_name>替换成实际的值

```
eksctl utils associate-iam-oidc-provider --cluster <cluster_name> --approve
```

### 2 安装Amazon Load Balancer Controller

2.1 配置集群和区域等参数，根据实际环境进行替换 

```
export AWS_REGION=cn-northwest-1
export CLUSTER_NAME=test
```

2.2 下载Amazon Load Balancer Controller所需的IAM策略文件

```
curl -o iam_policy_cn.json https://raw.githubusercontent.com/kubernetes-sigs \
/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy_cn.json
```
2.3 使用该策略文件创建IAM策略

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json
```
2.4 记录返回的Policy ARN

```
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName \
==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
```
2.5 创建aws-load-balancer-controller所需的IAM Role

```
eksctl create iamserviceaccount \
                            --cluster=${CLUSTER_NAME} \
                            --namespace=kube-system \
                            --name=aws-load-balancer-controller \
                            --attach-policy-arn=${POLICY_NAME} \
                            --override-existing-serviceaccounts \
                            --approve          
```
2.6 安装cert-manager

```
kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.1.1/cert-manager.yaml
```
2.7 下载controller定义文件

```
curl -o v2_2_0_full.yaml https://raw.githubusercontent.com/kubernetes-sigs \
/aws-load-balancer-controller/v2.2.0/docs/install/v2_2_0_full.yaml
```
2.8 将v2_2_0_full.yaml中的CLUSTER_NAME替换成实际环境的值

```
spec:
      containers:
        - args:
            - --cluster-name=CLUSTER_NAME
            - --ingress-class=alb
```
2.9 创建aws-load-balancer-controller

```
kubectl apply -f v2_2_0_full.yaml
```
2.10 查看Deployment部署状态

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
输出参考

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s
```

### 3 部署2048示例应用程序进行测试
3.1 下载定义文件，并部署2084示例应用程序

```
wget https://raw.githubusercontent.com/kubernetes-sigs/ \
aws-load-balancer-controller/v2.2.0/docs/examples/2048/2048_full.yaml

kubectl apply -f 2048_full.yaml
```
3.2 查看ALB Ingress状态

```
kubectl get ingress/ingress-2048 -n game-2048
```
输出参考

```
NAME           CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
ingress-2048   <none>   *       k8s-game2048-ingress2-xxxxxxxxxx-yyyyyyyyyy.us-west-2.elb.amazonaws.com   80      2m32s
```
3.3 复制2.2步骤中返回的ALB Ingress ADDRESS，粘贴到浏览器中进行访问

3.4 测试完成后，执行如下命令清理示例应用程序

```
kubectl delete -f 2048_full.yaml
```
