# Deploy the Kubernetes Ingress Controller 
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong/kong --generate-name --set ingressController.installCRDs=false

# Setup environment variables
kcx

# Testing connectivity to Kong Gateway
curl -i $PROXY_IP


# Deploy an upstream HTTP application 
kubectl apply -f https://bit.ly/echo-service

# Create a configuration group
echo "
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: kong
spec:
  controller: ingress-controllers.konghq.com/kong
" | kubectl apply -f -

# Add routing configuration
echo "
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: echo
  annotations:
    konghq.com/strip-path: 'true'
spec:
  ingressClassName: kong
  rules:
  - host: kong.example
    http:
      paths:
      - path: /echo
        pathType: ImplementationSpecific
        backend:
          service:
            name: echo
            port:
              number: 80
" | kubectl apply -f -

# test routing rule   
sudo echo "10.107.5.197   kong.example " >> /etc/hosts
curl -i http://kong.example/echo

# Add authentication to the service
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: example-auth
plugin: key-auth
" | kubectl apply -f -

# associate this plugin with the previous Ingress rule we created using the konghq.com/plugins annotation
kubectl annotate ingress echo konghq.com/plugins=example-auth

# will now require a valid API key
curl -si http://kong.example/echo --resolve kong.example:80:$PROXY_IP

# Provision a consumer and credential
kubectl create secret generic kotenok-key-auth \
  --from-literal=kongCredType=key-auth  \
  --from-literal=key=gav

#  create a KongConsumer resource that uses the Secret
echo "apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: kotenok
  annotations:
    kubernetes.io/ingress.class: kong
username: kotenok
credentials:
- kotenok-key-auth
" | kubectl apply -f -

# Use the credential
curl -si http://kong.example/echo --resolve kong.example:80:$PROXY_IP -H "apikey: gav"
