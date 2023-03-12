# init cluster
minikube start --driver=hyperkit --cpus=6 --memory=18432

# install kong kubernetes ingress controller
kubectl create -f https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/v2.8.1/deploy/single/all-in-one-dbless.yaml

using helm
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong/kong --generate-name --set ingressController.installCRDs=false

# get PROXY_IP
export PROXY_IP=$(minikube service -n kong kong-proxy --url | head -1)
# If installed by helm, service name would be "<release-name>-kong-proxy".
# $ export PROXY_IP=$(minikube service <release-name>-kong-proxy --url | head -1)
echo $PROXY_IP
http://192.168.64.161:32163

# test connectivity
curl -i $PROXY_IP

# deploy demo services
kubectl apply -f kubernetes/backToFuture/
kubectl apply -f kubernetes/chuck/

# test ingress
curl -i $PROXY_IP/back2future 
curl -i $PROXY_IP/chuck

