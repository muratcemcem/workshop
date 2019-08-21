

## Requirements

```
sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl

sudo yum -y install jq gettext
```

# cloud9 settings

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')

echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" >> ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```

# ssh file generate
```
ssh-keygen
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub

```
# iam control
```
aws sts get-caller-identity
```

# eksctl install
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```



## create eks cluster 
```
eksctl create cluster --version=1.13 --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=eu-west-1
```

# helm install
```
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod +x get_helm.sh
./get_helm.sh

kubectl apply -f helm/rbac.yaml
helm init --service-account tiller
```

# Efk installation

```
kubectl apply -f efk/.
kubectl get pods --namespace=kube-system
kubectl -n kube-system  port-forward $(kubectl -n kube-system get pod -l k8s-app=kibana-logging -o jsonpath='{.items[0].metadata.name}') 8080:5601
```

# install flagger nginx
```
helm repo add flagger https://flagger.app

helm upgrade -i nginx-ingress stable/nginx-ingress \
--namespace ingress-nginx \
--set controller.stats.enabled=true \
--set controller.metrics.enabled=true \
--set controller.podAnnotations."prometheus\.io/scrape"=true \
--set controller.podAnnotations."prometheus\.io/port"=10254

helm upgrade -i flagger flagger/flagger \
--namespace ingress-nginx \
--set prometheus.install=true \
--set meshProvider=nginx
kubectl create namespace test
kubectl -n test apply -f flagger/loadtester/.
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml

helm delete --purge flagger

```
# Canary example
```
kubectl -n test apply -f flagger/nginx/.

kubectl -n test apply -f flagger/podinfo-ingress.yaml
kubectl -n test apply -f flagger/podinfo-canary.yaml
kubectl apply -f https://raw.githubusercontent.com/weaveworks/flagger/master/artifacts/flagger/crd.yaml
kubectl -n test set image deployment/podinfo podinfod=stefanprodan/podinfo:2.0.1
kubectl -n  ingress-nginx port-forward flagger-prometheus-844dfd8f6f-lwr44 8080:909
kubectl -n test describe canary/podinfo
watch kubectl get canaries --all-namespaces

```

# istioctl install
```
curl -L https://git.io/getLatestIstio | sh -
cd istio-*

sudo mv -v bin/istioctl /usr/local/bin/

helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system

kubectl apply -f ../kiali/kiali.yaml # admin admin

helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set global.configValidation=false --set sidecarInjectorWebhook.enabled=false --set grafana.enabled=true --set servicegraph.enabled=true --set kiali.enabled=true

kubectl create namespace test-istio
kubectl label namespace  test-istio  istio-injection=enabled
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 8080:20001
kubectl apply -f demo-istio/.
kubectl -n test-istio port-forward svc/demo-frontend 8080:5000
```


# clean up
```
eksctl delete cluster --name=eksworkshop-eksctl
```

# references
https://eksctl.io
https://helm.sh
https://docs.flagger.app
https://istio.io
