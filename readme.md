```
# install istio
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY

# create nginx config map 
kubectl create configmap egress-sni-proxy-configmap -n istio-system --from-file=nginx.conf=./sni-proxy.conf -o yaml --dry-run=client | kubectl apply -f -

# deploy egress gateway with sni proxy
kubectl apply -f ./egressgateway-with-sni-proxy.yaml

# deploy resources
kubectl apply -f ./egress-resources.yaml -n istio-system

# deploy test app
kubectl apply -f <(istioctl kube-inject -f ./nginx.yaml)
export SOURCE_POD=$(kubectl get pod -l app=nginx -o jsonpath={.items..metadata.name})

# test
kubectl exec "$SOURCE_POD" -c sleep -- sh -c 'curl -v https://en.wikipedia.org/wiki/Main_Page'

# istio-proxy log - gateway
kubectl logs -l istio=egressgateway-with-sni-proxy -c istio-proxy -n istio-system

# istio-proxy log - sidecar
kubectl logs "$SOURCE_POD" -c istio-proxy -n istio-system

# SNI proxy log
kubectl logs -l istio=egressgateway-with-sni-proxy -n istio-system -c sni-proxy
```

```
# Create a root certificate and private key to sign the certificate for your services:
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt

#Create a certificate and a private key for my-nginx.mesh-external.svc.cluster.local:
openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt

# Generate client certificate and private key:
openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt

# Create a namespace to represent services outside the Istio mesh, namely mesh-external. Note that the sidecar proxy will not be automatically injected into the pods in this namespace since the automatic sidecar injection was not enabled on it.
kubectl create namespace mesh-external

# Create Kubernetes Secrets to hold the server’s and CA certificates.
kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
kubectl create -n istio-system secret generic nginx-ca-certs --from-file=example.com.crt

# Create a Kubernetes ConfigMap to hold the configuration of the NGINX server:
kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf -o yaml --dry-run=client | kubectl apply -f -

# Deploy nginx server
kubectl apply -f ./nginx_server.yaml

# Create Kubernetes Secrets to hold the client’s certificates:
kubectl create secret -n istio-system generic client-credential --from-file=tls.key=client.example.com.key \
  --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt

kubectl create secret -n istio-system generic client-credential-cacert --from-file=cacert=example.com.crt

```
# Cert troubleshooting
```
istioctl proxy-config secret nginx-54d7c7dc6c-cvw89 -n istio-system
```