# sockshop-istio
Sockshop demo with Istio service mesh

## File details
|Files|Description|
|------|--------|
|Sock-shop-complete-demo.yaml| contains all the changes required in sock-shop deployment to be compatible with Istio.|
|sock-shop-complete-demo-istio-1.yaml| sock-shop deployment yaml spec with manually injected Istio|

## Installation/Setup                                                                                                                                 

1. Install Istio-1.0.4 using helm charts.
2. Run bellow command due for `catalogue` service to be able to connect to `catalogue-db`. More info : https://github.com/istio/istio/issues/10062

```                                                                                                                                                   
  kubectl delete meshpolicies.authentication.istio.io default
```
3. Deploy sock-shop application.

```                                                                                                                                                   
  kubectl apply -f sock-shop-complete-demo-istio-1.yaml
```

## Circuit Breaker Pattern

1. Run Fortio app with 3 connections and 20 requests. See all requests go through
```
kubectl apply -f 4-fortio-deploy-istio.yaml
FORTIO_POD=$(kubectl get pod -nsock-shop | grep fortio | awk '{ print $1 }')
kubectl exec -it $FORTIO_POD  -nsock-shop -c fortio /usr/local/bin/fortio -- load -curl  http://front-end:80/index.html
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
2. Apply circuit breaker destination rule for max 1 connection.
```
kubectl apply -f 4-circuit-breaker.yaml
```
2. Run Fortio app with 3 connections and 20 requests. 30% should pass and 70% should fail.
```
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
3. Update destination rule for max 2 concurrent connections.
4. Run Fortio app with 3 connections and 20 requests. 70% should pass and 30% should fail.
```
kubectl exec -it $FORTIO_POD -nsock-shop -c fortio /usr/local/bin/fortio -- load -c 3 -qps 0 -n 20 -loglevel Warning http://front-end:80/index.html
```
  
## Security - Mutual TLS

1. Apply mesh-wide authentication policy in `default` namespace. This will enable all the receiving (server) sides of the service to use TLS.
```
istioctl create -f 5-global-mtls-mesh-policy.yaml
```
2. Load the front-end in browser. See that catalogues are not loading since `catalogue` service is rejecting plain-text `front-end` connections.
3. Update all the destination-rules to use TLS. This will enable all the sender (client) sides of the services to use TLS.
```
istioctl replace -f 5-virtual-services-all-mtls.yaml
```
4. Load the front-end again. See that its fuctioning properly now.
