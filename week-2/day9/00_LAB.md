
-----------------------------------------------------
AKS setup
-----------------------------------------------------

az group create --name nag-rg --location centralindia

az aks create \
    --resource-group nag-rg \
    --name nag-aks \
    --generate-ssh-keys \
    --node-count 3 \
    --zones 1 2 3

kubectl get nodes -o wide
kubectl get nodes --show-labels

az aks delete -n nag-aks --resource-group nag-rg --yes

az group delete -n nag-rg --yes

-----------------------------------------------------
ingress-nginx setup on AKS
-----------------------------------------------------

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --create-namespace \
  --namespace kafka \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz

kubectl get services --namespace kafka -o wide ingress-nginx-controller

-----------------------------------------------------
🛑 update ingress-ngix's external-ip DNS records
-----------------------------------------------------

-----------------------------------------------------
SSL Passthrough
-----------------------------------------------------

kubectl get deployments --all-namespaces | grep ingress-nginx
kubectl edit deployment ingress-nginx-controller -n kafka

Add the SSL Passthrough Flag
```yaml
spec:
  template:
    spec:
      containers:
      - args:
        - /nginx-ingress-controller
        - --publish-service=$(POD_NAMESPACE)/ingress-nginx-controller
        - --election-id=ingress-controller-leader
        - --ingress-class=nginx
        - --configmap=$(POD_NAMESPACE)/ingress-nginx-controller
        - --validating-webhook=:8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        - --default-ssl-certificate=$(POD_NAMESPACE)/tls-secret
        - --enable-ssl-passthrough
        - --v=2
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.0
```

kubectl rollout status deployment ingress-nginx-controller -n kafka
kubectl describe deployment ingress-nginx-controller -n kafka


-----------------------------------------------------
rack-awareness
-----------------------------------------------------

./docs/rack-awareness.md

https://docs.google.com/presentation/d/1pMTUxbKQqLo9Dq_oqL-ZSaXo8PU1cQPndUyQwGgox3w/edit#slide=id.g266c580a345_1_55


-----------------------------------------------------
Affinity
-----------------------------------------------------

./docs/affinity.md
https://docs.google.com/presentation/d/1pMTUxbKQqLo9Dq_oqL-ZSaXo8PU1cQPndUyQwGgox3w/edit#slide=id.g268704068db_0_4



k get nodes -o wide
kubectl taint nodes aks-nodepool1-32500729-vmss000000 dedicated=Kafka:NoSchedule
kubectl taint nodes aks-nodepool1-32500729-vmss000001 dedicated=Kafka:NoSchedule
kubectl taint nodes aks-nodepool1-32500729-vmss000002 dedicated=Kafka:NoSchedule

kubectl label nodes aks-nodepool1-32500729-vmss000000 dedicated=Kafka
kubectl label nodes aks-nodepool1-32500729-vmss000001 dedicated=Kafka
kubectl label nodes aks-nodepool1-32500729-vmss000002 dedicated=Kafka




-----------------------------------------------------
Deploy Strimzi Operator(s)
-----------------------------------------------------

Deploying the Cluster Operator

kubectl create namespace kafka
sed -i 's/namespace: .*/namespace: kafka/' ./strimzi-0.38.0/install/cluster-operator/*RoleBinding*.yaml
kubectl create -f ./strimzi-0.38.0/install/cluster-operator -n kafka
kubectl get deployment -n kafka
kubectl get pods -n kafka -o wide
kubectl logs deployment/strimzi-cluster-operator -n kafka 
kubectl delete -f ./strimzi-0.38.0/install/cluster-operator -n kafka

🛑 single replica 'cluster-operator' is not safe for production, recommended 2

https://docs.google.com/presentation/d/1pMTUxbKQqLo9Dq_oqL-ZSaXo8PU1cQPndUyQwGgox3w/edit#slide=id.g268704068db_0_101


-----------------------------------------------------
Resources
-----------------------------------------------------

./doc/resources.md
https://docs.google.com/presentation/d/1pMTUxbKQqLo9Dq_oqL-ZSaXo8PU1cQPndUyQwGgox3w/edit#slide=id.g268764217b2_0_0



-----------------------------------------------------
Storage, How Many Disks? ,Partition Rebalance
-----------------------------------------------------


https://docs.google.com/presentation/d/1pMTUxbKQqLo9Dq_oqL-ZSaXo8PU1cQPndUyQwGgox3w/edit#slide=id.g268764217b2_0_105


-----------------------------------------------------
Broker's Listeners
-----------------------------------------------------

 Types of listeners

  - PLAINTEXT
  - TLS
  - LOADBALANCER
  - NODEPORT
  - ROUTE
  - INGRESS

  When to use which listener type?
  - PLAINTEXT: for internal communication
  - TLS: for external communication
  - EXTERNAL: for external communication 
  - LOADBALANCER: for external communication 
  - NODEPORT: for external communication 
  - ROUTE: for external communication (OpenShift)
  - INGRESS: for external communication (Kubernetes)


-----------------------------------------------------

<!-- kubectl delete -f ./kafka_v1.yaml -n kafka -->
kubectl apply -f ./kafka_v2.yaml -n kafka
kubectl get pod -o wide -n kafka


kubectl get pod -n kafka
kubectl get svc -n kafka
kubectl get ingress -n kafka

kubectl describe ingress my-cluster-kafka-bootstrap -n kafka -o yaml

-----------------------------------------------------
Create truststore from CA certificate
-----------------------------------------------------  

./docs/truststore.md

-----------------------------------------------------
Test with Kafka Client ( Producer  ) ( java based )
-----------------------------------------------------  

🤚

-----------------------------------------------------
Metrics - kafka-Exporter, Prometheus, Grafana
-----------------------------------------------------


**Updating kafka Custom Resource for Kafka Exporter & Prometheus Monitoring**


**Deploying the Prometheus Operator**


curl -s https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml > ./metrics/prometheus-operator-deployment.yaml
sed -E -i '/[[:space:]]*namespace: [a-zA-Z0-9-]*$/s/namespace:[[:space:]]*[a-zA-Z0-9-]*$/namespace: kafka/' ./metrics/prometheus-operator-deployment.yaml
kubectl create -f ./metrics/prometheus-operator-deployment.yaml -n kafka
kubectl get pods -n kafka


**Deploying Prometheus**

sed -i 's/namespace: .*/namespace: kafka/' ./metrics/prometheus-install/prometheus.yaml
kubectl apply -f ./metrics/prometheus-additional-properties/prometheus-additional.yaml -n kafka
kubectl apply -f ./metrics/prometheus-install/strimzi-pod-monitor.yaml -n kafka
kubectl apply -f ./metrics/prometheus-install/prometheus-rules.yaml -n kafka
kubectl apply -f ./metrics/prometheus-install/prometheus.yaml -n kafka
kubectl get pods -n kafka -w


**Deploying Grafana**

kubectl apply -f ./metrics/grafana-install/grafana.yaml -n kafka
kubectl get service grafana -n kafka
<!-- kubectl port-forward svc/grafana 3000:3000 -n kafka -->
<!-- kubectl delete -f ./metrics/ingress-grafana.yaml -n kafka -->
<!-- kubectl get ingress ingress-grafana -n kafka -->



-----------------------------------------------------
Mirror Maker 2
-----------------------------------------------------

./docs/mirror-maker-2.md

