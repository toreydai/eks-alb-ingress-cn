## 中国区Amazon EKS中使用ALB
### 1 安装Amazon Load Balancer Controller

1.1 配置集群和区域等参数，根据实际环境进行替换 

```
export AWS_REGION=cn-northwest-1
export CLUSTER_NAME=test
```

1.2 下载Amazon Load Balancer Controller所需的IAM策略文件

```
curl -o iam_policy_cn.json https://raw.githubusercontent.com/kubernetes-sigs \
/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy_cn.json
```
1.2 使用该策略文件创建IAM策略

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json
```
1.3 记录返回的Policy ARN

```
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName \
==`AWSLoadBalancerControllerIAMPolicy`].Arn' --output text --region ${AWS_REGION})
```
1.4 创建aws-load-balancer-controller所需的IAM Role

```
eksctl create iamserviceaccount \
                            --cluster=${CLUSTER_NAME} \
                            --namespace=kube-system \
                            --name=aws-load-balancer-controller \
                            --attach-policy-arn=${POLICY_NAME} \
                            --override-existing-serviceaccounts \
                            --approve          
```
1.5 安装cert-manager

```
kubectl apply \
    --validate=false \
    -f https://github.com/jetstack/cert-manager/releases/download/v1.1.1/cert-manager.yaml
```
1.6 下载controller定义文件

```
curl -o v2_2_0_full.yaml https://raw.githubusercontent.com/kubernetes-sigs \
/aws-load-balancer-controller/v2.2.0/docs/install/v2_2_0_full.yaml
```
1.7 将v2_2_0_full.yaml中的CLUSTER_NAME替换成实际环境的值

```
spec:
      containers:
        - args:
            - --cluster-name=CLUSTER_NAME
            - --ingress-class=alb
```
1.8 创建aws-load-balancer-controller

```
kubectl apply -f v2_2_0_full.yaml
```
1.9 查看Deployment部署状态

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```
输出参考

```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           84s
```

### 2 部署2048示例应用程序进行测试
2.1 下载定义文件，并部署2084示例应用程序

```
wget https://raw.githubusercontent.com/kubernetes-sigs/ \
aws-load-balancer-controller/v2.2.0/docs/examples/2048/2048_full.yaml

kubectl apply -f 2048_full.yaml
```
2.2 查看ALB Ingress状态

```
kubectl get ingress/ingress-2048 -n game-2048
```
输出参考

```
NAME           CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
ingress-2048   <none>   *       k8s-game2048-ingress2-xxxxxxxxxx-yyyyyyyyyy.us-west-2.elb.amazonaws.com   80      2m32s
```
2.3 复制2.2步骤中返回的ALB Ingress ADDRESS，粘贴到浏览器中进行访问

2.4 测试完成后，执行如下命令清理示例应用程序

```
kubectl delete -f 2048_full.yaml
```
