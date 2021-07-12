# Integrating Secrets Manager secrets with Kubernetes Secrets Store CSI Driver Notes

## Reference for Setup

-<https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html>

## Install

```shell
helm repo add secrets-store-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/secrets-store-csi-driver/master/charts
helm -n kube-system install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set grpcSupportedProviders="aws"

kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

kubectl get pods -n kube-system
```

### Setup IAM role for IRSA (`iamserviceaccount`)

```shell
aws --region ${EKS_REGION} secretsmanager  create-secret \
    --name MySecret --secret-string '{"username":"foo", "password":"bar"}'

aws iam create-policy \
    --policy-name SecretsManagerK8SPolicy \
    --policy-document file://resources/aws/SecretsManagerK8SPolicy.json

export AWS_ACCOUNT=$(aws sts get-caller-identity --output text --query 'Account')
export EKS_REGION="us-east-1"
export CLUSTER_NAME="<your_cluster_name>"
export NAMESPACE="dev"

eksctl create iamserviceaccount \
    --name nginx-deployment-sa \
    --namespace ${NAMESPACE} \
    --region ${EKS_REGION} \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn "arn:aws:iam::${AWS_ACCOUNT}:policy/SecretsManagerK8SPolicy" \
    --approve \
    --override-existing-serviceaccounts

eksctl get iamserviceaccount --cluster $CLUSTER_NAME --namespace ${NAMESPACE}
eksctl get iamserviceaccount nginx-deployment-sa --cluster $CLUSTER_NAME --namespace ${NAMESPACE}
```

```shell
kubectl apply -f secret-provider-class.yaml -n ${NAMESPACE}
kubectl apply -f pod.yaml -n ${NAMESPACE}

kubectl describe pod nginx-deployment -n ${NAMESPACE}
```

## Test

```shell
kubectl exec -it nginx-deployment -n ${NAMESPACE} -- cat /mnt/secrets-store/MySecret; echo
```
