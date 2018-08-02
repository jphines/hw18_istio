# HW 2018 k8s & isitio

## test-service
This sample app in `samples/test-service` is my own noodling on circuit breakers
within istio.

There are two different docker containers for this service, tagged `200` and `503` which
return their tagged status codes. Source can be found at https://github.com/jphines/test-service.

We setupu the `fortio` client used to load test istio as our demo driver. See setup https://github.com/istio/istio/tree/master/samples/httpbin/sample-client.


### -all
The `-all` configuration creates a load balancing pool with both 200 and 503 apps. It is specifically
sized such that the 200 apps are greater than 50% of the pool. This is to avoid the default envoy panic
threshold during outlier ejection. https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/outlier
and https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/load_balancing#arch-overview-load-balancing-panic-threshold.

```
bash-3.2$ kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 20 -n 200 -loglevel Warning http://test-service:8080/
. . .
Code 200 : 195 (97.5 %)
Code 503 : 5 (2.5 %)
```

Example output of fortio load test with `-all`

### regular
The non specified configuration creates a load balancing pool with two distinct subsets of both the 200 and 503 apps.
A load test now will trigger a load balancing panic which will cause envoy to load balance between all hosts in 
the failing 503 subset.

```
bash-3.2$ kubectl exec -it $FORTIO_POD  -c fortio /usr/local/bin/fortio -- load -c 20 -n 200 -loglevel Warning http://test-service:8080/
Code 200 : 136 (75.6 %)
Code 503 : 44 (24.4 %)
```

We can verify the panic behavior by checking the prometheus metrics on the outbound container
```
bash-3.2$ kubectl exec -it $FORTIO_POD  -c istio-proxy  -- sh -c 'curl localhost:15000/stats/prometheus' | grep panic | grep -v TYPE
envoy_cluster_outbound|8080|503|test_service_default_svc_cluster_local_lb_healthy_panic{} 45
```

This metrics will also show in prometheus if configured.
