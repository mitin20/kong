# Deploy the Kubernetes Ingress Controller 
helm repo add kong https://charts.konghq.com
helm repo update
helm install kong/kong --generate-name --set ingressController.installCRDs=false

# Setup environment variables
HOST=$(kubectl get svc --namespace default kong-1678801086-kong-proxy -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
PORT=$(kubectl get svc --namespace default kong-1678801086-kong-proxy -o jsonpath='{.spec.ports[0].port}')
export PROXY_IP=${HOST}:${PORT}
curl $PROXY_IP

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
sudo echo "10.102.54.122   kong.example " >> /etc/hosts
curl -i http://kong.example/echo

# Add TLS configuration
openssl req -subj '/CN=kong.example' -new -newkey rsa:2048 -sha256 \
  -days 365 -nodes -x509 -keyout server.key -out server.crt \
  -extensions EXT -config <( \
   printf "[dn]\nCN=kong.example\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:kong.example\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth") 2>/dev/null;
  openssl x509 -in server.crt -subject -noout

# Then, create a Secret containing the certificate
kubectl create secret tls kong.example --cert=./server.crt --key=./server.key

# Finally, update your routing configuration to use this certificate:
kubectl patch --type json ingress echo -p='[{
    "op":"add",
	"path":"/spec/tls",
	"value":[{
        "hosts":["kong.example"],
		"secretName":"kong.example"
    }]
}]'

# test certificate
curl -ksv https://kong.example/echo --resolve kong.example:443:$PROXY_IP 2>&1 | grep -A1 "certificate:"

# Configuring ACL Plugin
# Add routing configuration
echo "
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lemon
  annotations:
    konghq.com/strip-path: 'true'
spec:
  ingressClassName: kong
  rules:
  - host: kong.example
    http:
      paths:
      - path: /lemon
        pathType: ImplementationSpecific
        backend:
          service:
            name: echo
            port:
              number: 80
" | kubectl apply -f -

# Test the routing rule
curl -i http://kong.example/lemon --resolve kong.example:80:$PROXY_IP

# create a second pointing to the same Service
echo "
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lime
  annotations:
    konghq.com/strip-path: 'true'
spec:
  ingressClassName: kong
  rules:
  - host: kong.example
    http:
      paths:
      - path: /lime
        pathType: ImplementationSpecific
        backend:
          service:
            name: echo
            port:
              number: 80
" | kubectl apply -f -

# Test the routing rule
curl -i http://kong.example/lime --resolve kong.example:80:$PROXY_IP

# Add JWT authentication to the service
echo "
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: app-jwt
plugin: jwt
" | kubectl apply -f -

# associate the plugin to the Ingress rules we created earlier
kubectl annotate service echo konghq.com/plugins=app-jwt

# Requests without credentials are now rejected
curl -si http://kong.example/lemon --resolve kong.example:80:$PROXY_IP
HTTP/1.1 401 Unauthorized

# Provision consumers
echo "apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: admin
  annotations:
    kubernetes.io/ingress.class: kong
username: admin
" | kubectl apply -f -

echo "apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: user
  annotations:
    kubernetes.io/ingress.class: kong
username: user
" | kubectl apply -f -

# Create Secrets

kubectl create secret \
  generic admin-jwt  \
  --from-literal=kongCredType=jwt  \
  --from-literal=key="admin-issuer" \
  --from-literal=algorithm=RS256 \
  --from-literal=rsa_public_key="-----BEGIN PUBLIC KEY-----

-----END PUBLIC KEY-----"

kubectl create secret \
  generic user-jwt  \
  --from-literal=kongCredType=jwt  \
  --from-literal=key="user-issuer" \
  --from-literal=algorithm=RS256 \
  --from-literal=rsa_public_key="-----BEGIN PUBLIC KEY-----

-----END PUBLIC KEY-----"

# Assign the credentials
kubectl patch --type json kongconsumer admin \
  -p='[{
    "op":"add",
    "path":"/credentials",
    "value":["admin-jwt"]
  }]'

kubectl patch --type json kongconsumer user \
  -p='[{
    "op":"add",
    "path":"/credentials",
    "value":["user-jwt"]
  }]'

# Send authenticated requests
export ADMIN_JWT=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJhZG1pbi1pc3VlciJ9.

export USER_JWT=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ1c2VyLWlzdWVyIn0.

curl -I -H "Authorization: Bearer ${USER_JWT}" http://kong.example/lemon --resolve kong.example:80:$PROXY_IP

curl -ki -vvv http://kong.example/lemon -H "Authorization: Bearer ${USER_JWT}"