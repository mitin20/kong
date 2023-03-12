# create secret for jwt public key
```
kubectl create secret \
  generic app-admin-jwt  \
  --from-literal=kongCredType=jwt  \
  --from-literal=key="admin-issuer" \
  --from-literal=algorithm=RS256 \
  --from-literal=rsa_public_key="-----BEGIN PUBLIC KEY-----
  MIIBIjA....
  -----END PUBLIC KEY-----"
```

# create a second secret with a different key
```
kubectl create secret \
  generic app-user-jwt  \
  --from-literal=kongCredType=jwt  \
  --from-literal=key="user-issuer" \
  --from-literal=algorithm=RS256 \
  --from-literal=rsa_public_key="-----BEGIN PUBLIC KEY-----
  qwerlkjqer....
  -----END PUBLIC KEY-----"
```

# Now let’s associate the plugin to the Ingress rules we created earlier.
```
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-get
  annotations:
    konghq.com/plugins: app-jwt
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /get
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-post
  annotations:
    konghq.com/plugins: app-jwt
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /post
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -
```


# create secrets for acl groups
```
kubectl create secret \
  generic app-admin-acl \
  --from-literal=kongCredType=acl  \
  --from-literal=group=app-admin
kubectl create secret \
  generic app-user-acl \
  --from-literal=kongCredType=acl  \
  --from-literal=group=app-user
```

# The last thing to configure is the ingress to use the new plguins. Note, if you set more than one ACL plugin, the last one supplied will be the only one evaluated.
```
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-get
  annotations:
    konghq.com/plugins: app-jwt,plain-user-acl
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /get
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -
echo '
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-post
  annotations:
    konghq.com/plugins: app-jwt,admin-acl
    konghq.com/strip-path: "false"
    kubernetes.io/ingress.class: kong
spec:
  rules:
  - http:
      paths:
      - path: /post
        backend:
          serviceName: httpbin
          servicePort: 80
' | kubectl apply -f -
```
