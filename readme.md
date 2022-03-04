```
# install istio
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY

# Create a root certificate and private key to sign the certificate for your services:
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

#Create a certificate and a private key for my-nginx.mesh-external.svc.cluster.local:
openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt

# Generate client certificate and private key:
openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt

# Create Kubernetes Secrets to hold the client’s certificates:
kubectl create secret generic client-credential --from-file=tls.key=client.example.com.key \
  --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt

# deploy test app
kubectl apply -f <(istioctl kube-inject -f ./nginx.yaml)
export SOURCE_POD=$(kubectl get pod -l app=nginx -o jsonpath={.items..metadata.name})

# deploy resources
kubectl apply -f ./egress-resources-direct.yaml -n istio-system

# test
kubectl exec "$SOURCE_POD" -c sleep -- sh -c 'curl -vk https://certauth.idrix.fr'


```
# Troubleshooting
```
istioctl proxy-config secret nginx-54d7c7dc6c-cvw89 -n istio-system
istioctl proxy-config cluster istio-egressgateway-with-sni-proxy-74f5b6c746-zl696 -n istio-system
istioctl proxy-config listener istio-egressgateway-with-sni-proxy-74f5b6c746-zl696 -n istio-system
istioctl proxy-config cluster nginx-54d7c7dc6c-6psfk -n istio-system
istioctl proxy-config listener nginx-54d7c7dc6c-6psfk -n istio-system
```