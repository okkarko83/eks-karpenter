1\. Export env for tempory output for karpenter cloud formation.

```sh
export TEMPOUT=$(mktemp)
```

2\. Create cloud formation stack for karpenter

```sh
curl -fsSL https://karpenter.sh/"v0.27.0"/getting-started/getting-started-with-eksctl/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-{YOUR_CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName={YOUR_CLUSTER_NAME}"
```

3.Create eks cluster

```sh
eksctl create cluster -f karpenterdemo.yaml
```

4.Export cluster end point

```sh
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name varcamp-playground --query "cluster.endpoint" --output text)"
```

5.Export Karpenter IAM Role ARN

```sh
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::{YOUR_AWS_ACCOUNT_ID}:role/{YOUR_CLUSTER_NAME}-karpenter"
```

6.Check ENV

```sh
echo $CLUSTER_ENDPOINT $KARPENTER_IAM_ROLE_ARN
```

7.Check IAM Role

```sh
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true
# If the role has already been successfully created, you will see:
# An error occurred (InvalidInput) when calling the CreateServiceLinkedRole operation: Service role name AWSServiceRoleForEC2Spot has been taken in this account, please try a different suffix.
```

8.Install Karpenter with helm

```sh
helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version v0.27.0 --namespace karpenter --create-namespace \
    --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::{YOUR_AWS_ACCOUNT_ID}:role/KarpenterNodeRole-{YOUR_AWS_ACCOUNT_ID}" \
    --set settings.aws.clusterName={YOUR_CLUSTER_NAME} \
    --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-{YOUR_CLUSTER_NAME} \
    --set settings.aws.interruptionQueueName={YOUR_CLUSTER_NAME} \
    --wait
```

9.Install Prometheus and grafana

```sh
helm repo add grafana-charts https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

curl -fsSL https://karpenter.sh/"0.27.0"/getting-started/getting-started-with-eksctl/prometheus-values.yaml | tee prometheus-values.yaml
helm install --namespace monitoring prometheus prometheus-community/prometheus --values prometheus-values.yaml

curl -fsSL https://karpenter.sh/"0.27.0"/getting-started/getting-started-with-eksctl/grafana-values.yaml | tee grafana-values.yaml
helm install --namespace monitoring grafana grafana-charts/grafana --values grafana-values.yaml
```

10\. Create Provisioner

```sh
kubectl create -f provisioner_filename.yaml
```

