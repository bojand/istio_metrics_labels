# Instructions

Create a cluster.

If needed ensure that nodes are open to public ingress for HTTP @ 80 and HTTPS @ 433 and ICMP.

Install Istio

```sh
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.13.2 sh -
```

Create the configuration profile

```sh
./istio-1.13.2/bin/istioctl profile dump minimal > istio.minimal.yaml
```

Set `ingressGateways` to `true`.

```sh
./istio-1.13.2/bin/istioctl install -f ./istio.minimal.yaml
kubectl apply -f istio.gateway.yaml
```

```sh
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
```

Create the app

```sh
kubectl apply -f app.namespace.yaml
kubectl apply -f app.deployment.yaml
kubectl apply -f app.svc.yaml
kubectl apply -f app.vs.yaml
```

Test

```sh
http http://$GATEWAY_URL/status\?lag\=100 Host:sample-golang.example.com
http http://$GATEWAY_URL/headers Host:sample-golang.example.com
http http://$GATEWAY_URL/echo Host:sample-golang.example.com
```

Get metrics 

```sh
kubectl exec "$(kubectl get pod -n istio-system -l=istio=ingressgateway -o jsonpath='{.items[0].metadata.name}')" -n istio-system -c istio-proxy -- curl -sS 'localhost:15000/stats/prometheus' | grep istio_requests_total
```

Notice how `destination_app` is `unknown`.

```sh
total
# TYPE istio_requests_total counter
istio_requests_total{response_code="200",reporter="source",source_workload="istio-ingressgateway",source_workload_namespace="istio-system",source_principal="unknown",source_app="istio-ingressgateway",source_version="unknown",source_cluster="Kubernetes",destination_workload="sample-golang",destination_workload_namespace="sample-golang",destination_principal="unknown",destination_app="unknown",destination_version="unknown",destination_service="sample-golang.sample-golang.svc.cluster.local",destination_service_name="sample-golang",destination_service_namespace="sample-golang",destination_cluster="Kubernetes",request_protocol="http",response_flags="-",grpc_response_status="",connection_security_policy="unknown",source_canonical_service="istio-ingressgateway",destination_canonical_service="sample-golang",source_canonical_revision="latest",destination_canonical_revision="latest"} 3
```