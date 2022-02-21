```
# install istio
istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY

# create nginx config map 
kubectl create configmap egress-sni-proxy-configmap -n istio-system --from-file=nginx.conf=./sni-proxy.conf

# deploy egress gateway with sni proxy
kubectl apply -f ./egressgateway-with-sni-proxy.yaml

# deploy resources
kubectl apply -f ./egress-resources.yaml

# deploy test app
kubectl apply -f <(istioctl kube-inject -f ./sleep.yaml)
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath={.items..metadata.name})

# test
kubectl exec "$SOURCE_POD" -c sleep -- sh -c 'curl -s https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>.*</title>"; curl -s https://de.wikipedia.org/wiki/Wikipedia:Hauptseite | grep -o "<title>.*</title>"'

# istio-proxy log
kubectl logs -l istio=egressgateway-with-sni-proxy -c istio-proxy -n istio-system

# SNI proxy log
kubectl logs -l istio=egressgateway-with-sni-proxy -n istio-system -c sni-proxy



```