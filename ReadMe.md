# 免责说明
<br> 建议测试过程中使用此文档，生产环境使用请自行考虑评估。
<br> 当您对方案需要进一步的沟通和反馈后，可以联系 taodai@nwcdcloud.cn 获得更进一步的支持。
<br> 欢迎联系参与方案共建和提交方案需求，也欢迎在 github 项目issue中留言反馈bugs。

# 配置使用EBS CSI

1 准备工作
> 将本项目Clone到本地
```
git clone https://github.com/toreydai/aws-ebs-csi-driver-cn.git
```


2 创建所需要的IAM policy , EKS OIDC provider, service account

> 2.1 创建所需要的IAM policy
[https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json](https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json)

```bash

AWS_REGION=cn-northwest-1

aws iam create-policy \
    --policy-name Amazon_EBS_CSI_Driver \
    --policy-document file://./resource/ebs-csi-iam-policy.json \
    --region ${AWS_REGION}
        
#返回示例,请记录返回的Plociy ARN
POLICY_NAME=$(aws iam list-policies --query 'Policies[?PolicyName==`Amazon_EBS_CSI_Driver`].Arn' --output text --region ${AWS_REGION})
```

> 2.2 获取EKS工作节点的IAM role

```bash
# 注意这一步如果是多个nodegroup就会有多个role
kubectl -n kube-system describe configmap aws-auth

# 单个节点组
ROLE_NAME=Role-name-in-above-output
aws iam attach-role-policy --policy-arn ${POLICY_NAME} \
    --role-name ${ROLE_NAME} --region ${AWS_REGION}

#多个节点组, 这里准备了一个脚本updaterole.sh
sh aws-ebs-csi-driver/updaterole.sh ${POLICY_NAME}
```

> 2.3 部署EBS CSI 驱动到eks 集群

```bash
#git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git

#中国区请使用resource/aws-ebs-csi-driver的配置文件进行部署
kubectl apply -k resource/deploy/kubernetes/overlays/stable

# 验证部署正确
kubectl get pods -n kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
alb-ingress-controller-6f6d44869f-t7vz6   1/1     Running   0          25d
aws-node-qjpdm                            1/1     Running   0          25d
aws-node-zlc9f                            1/1     Running   0          6d18h
coredns-9dbbd6d65-n5tk5                   1/1     Running   0          25d
coredns-9dbbd6d65-v7lhc                   1/1     Running   0          25d
ebs-csi-controller-748845ccc-tc25c        4/4     Running   0          62m
ebs-csi-controller-748845ccc-wzl9f        4/4     Running   0          62m
ebs-csi-node-kkhfl                        3/3     Running   0          62m
ebs-csi-node-xhn4l                        3/3     Running   0          62m
efs-csi-node-7zlxb                        3/3     Running   0          6d18h
efs-csi-node-fkkgb                        3/3     Running   0          25d
kube-proxy-bmzmj                          1/1     Running   0          6d18h
kube-proxy-nfj9c                          1/1     Running   0          25d
metrics-server-7578984995-q28vn           1/1     Running   0          25d
```

3 部署动态卷实例应用

```bash
cd resource/examples/kubernetes/dynamic-provisioning/
kubectl apply -f specs/

#查看storageclass
kubectl describe storageclass ebs-sc

#查看示例app状态
kubectl get pods --watch
#查看是否有失败
kubectl get events

kubectl get pv
PV_NAME=$(kubectl get pv -o json | jq -r '.items[0].metadata.name')
kubectl describe persistentvolumes ${PV_NAME}

kubectl exec -it app cat /data/out.txt
# Wed Jan 6 03:05:06 UTC 2021
# Wed Jan 6 03:05:11 UTC 2021
# Wed Jan 6 03:05:16 UTC 2021
# Wed Jan 6 03:05:21 UTC 2021
# Wed Jan 6 03:05:26 UTC 2021
```
4 在EC2 Console中查看PV卷
![avatar](https://github.com/toreydai/aws-ebs-csi-driver-cn/blob/main/pv-ebs-gp3.png)

5 删除示例程序
kubectl delete -f specs/
