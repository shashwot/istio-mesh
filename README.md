# Istio And Kubernetes
Istio is a configurable, open source service-mesh layer that connects, monitors, and secures the containers in a Kubernetes cluster. Kubernetes manages availability and resource consumption of nodes, adding pods as demand increases with the pod autoscaler. Istio injects additional containers into the pod to add security, management, and monitoring.
Istio layers on top of Kubernetes, adding containers that are essentially invisible to the programmer and administrator. Called "sidecar" containers, these act as a "person in the middle," directing traffic and monitoring the interactions between components. The two work in combination in three ways: configuration, monitoring, and management.

# Service Mesh
The term service mesh is used to describe the network of microservices that make up applications and the interactions between them. As a service mesh grows in size and complexity, it can become harder to understand and manage. Its requirements can include discovery, load balancing, failure recovery, metrics, and monitoring. A service mesh also often has more complex operational requirements, like A/B testing, canary rollouts, rate limiting, access control, and end-to-end authentication.

# Why use istio
- Secure service-to-service communication in a cluster with TLS encryption, strong identity-based authentication and authorization
- Automatic load balancing for HTTP, gRPC, WebSocket, and TCP traffic
- Fine-grained control of traffic behavior with rich routing rules, retries, failovers, and fault injection
- A pluggable policy layer and configuration API supporting access controls, rate limits and quotas
- Automatic metrics, logs, and traces for all traffic within a cluster, including cluster ingress and egress

# Architecture

![istio-architecture.jpg](istio-architecture.jpg?raw=true "istio-architecture.jpg")


#### Proxy/Envoy
- It runs as a container inside each pod. Sidecar proxy per microservices handles inbound and outbound traffic within each pod.
#### Mixer
- Creates a policy/pre-condition check and telemetry.
#### Pilot
- Converts high level routing rules that controls traffic behaviour into envoy specific configs.
#### Citadel
- Certificate Authority for service to service authentication.

# Istio Glossary


![internal-arch.png](internal-arch.png?raw=true "internal-arch.png")

#### Gateway
- configures a load balancer for http/tcp traffic, enables ingress traffic into the service mesh
#### Virtual Service
- defines the rules that control how requests for a service are routed within a service mesh
#### Destination Rule
- configures the set of policies to be applied to the request after virtual service has occured.
#### Service Version (Subset)
- Subset allows to select a subset of pods based on the labels
#### Service Entry
- enables requests to services outside of the service mesh

# Installation

1. Go to the Istio Release page to download the installation files for your OS.
```
$ curl -L https://istio.io/downloadIstio | sh -
```
<br>
<hr>
<br>
2. Go to the directory

```
$ cd istio-1.10.0
```
<br>
<hr>
<br>

3. Add istioctl to the path

```
$ export PATH=$PWD/bin:$PATH
```
<br>
<hr>
<br>

4. Install istio in kubernetes cluster

```
istioctl install --set profile=demo -y
```


![istio-setup.png](istio-setup.png?raw=true "istio-setup.png")
<br>
<hr>
<br>
5. Add a namespace to instruct Istio to inject Envoy Proxy.

```
$ kubectl create namespace testing
$ kubectl label namespace testing istio-injection=enabled
$ kubectl describe namespace testing
```


![describe-ns.png](describe-ns.png?raw=true "describe-ns.png")
<br>
<hr>
<br>
6. Apply the deployment of BookInfo from the same istio sample.

```
$ kubectl apply -n testing -f samples/bookinfo/platform/kube/bookinfo.yaml
```
![envoy-inject.png](envoy-inject.png?raw=true "envoy-inject.png")

If you describe the pods, you will see that the envoy proxy has been injected into the pod.
<br>
<hr>
<br>
7.  Now we need to route the traffic to our service and pods. For this we need to create a gateway for http/tcp traffic to allow traffic into the service mesh.

**gateway.yml**
```
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: sampleapp-gateway
spec:
  selector:
    istio: ingressgateway # istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```
Apply the gateway configuration.
```
$ kubectl apply -f gateway.yml
```
<br>
<hr>
<br>
8. Lets create another file which will define a rule to control how the requets for a service are routed.

**virtual-service.yml**
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - sampleapp-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```
Apply the virtualservice configuration.
```
$ kubectl apply -f virtual-service.yml
```
We can check if the website is up and running using the following commands:
```
$ IP_LB=$(kubectl get svc istio-ingressgateway -n istio-system -o jsonpath='{.spec.clusterIP}')
$ curl -H "Host: xyz.com" $IP_LB/productpage
```
<br>
<hr>
<br>

9. We can define other rules where we can control how the behaviour of the services are routed within a mesh.

**virtual-service-all.yml**
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
    retries:    # configure 2 retries with 1s timeout and gateway-error,connect-failure,refused-stream
      attempts: 2
      perTryTimeout: 1s
      retryOn: gateway-error,connect-failure,refused-stream
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
        user-agent:
          regex: ".*Chrome.*"
        #cookie:
        #  regex: "^(.*?;)?(user=packtpub)(;.*)?$"
    route:
    - destination:
        host: reviews
        subset: v2
    timeout: 2s
  - route:
    - destination:
        host: reviews
        subset: v1
      #weight: 100
    mirror: 		#Shift Live Traffic to this subset for testing before going to production
      host: reviews
      subset: v3
    mirror_percent: 100
    timeout: 2s
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
```
Apply the virtualservice configuration.
```
$ kubectl apply -f virtual-service-all.yml
```
Here I have introduced the concept of circuit breaker, http header match with uri, timeouts, mirroring, weights, etc.

- Istio will retry to connect to the service if there are errors on gateway-error,connect-failure,refused-stream as shown in the productpage virutal service.
- Similarly, http match headers will redirect the traffic to subset v2 on reviews if the user logs in with the username jason and the user-aget(browser) as chrome.
- Mirrors are used to shift the live traffic to a certain subset for testing before going into production.
- Weights are used to redirect certain percentage of traffic into a certain pod. For example if the weight of the traffic in v1 is 80 and in v2 is 20, 80% of the traffic will flow into the v1 subset of the review.
- Timeouts are used to define the reply timeout if there is breakage in connection.

<br>
<hr>
<br>

10. Destination Rule
	It configures the set of policies to be applied to the requests after the Virtual Service has occured.
	
**destination-rule-all.yaml**
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
        connectTimeout: 1s
        tcpKeepalive:
          time: 3600s
          interval: 60s
      http:
       maxRetries: 3
       maxRequestsPerConnection: 50
       idleTimeout: 60s
       http1MaxPendingRequests: 25
       http2MaxRequests: 5
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  trafficPolicy:
    outlierDetection:
      consecutiveErrors: 5	# 5 upstream errors (502,503,504)
      interval: 30s		# Sliding window of 30s
      baseEjectionTime: 1m	# eject upstream for 1 min
      maxEjectionPercent: 50	# max 50% of upstream hosts ejected
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

```
Apply the destination-rule configuration.
```
$ kubectl apply -f destination-rule-all.yaml
```
Here we have set the destination rule with trafficPolicy to know the behaviour of the service mesh.

<br>
<hr>
<br>

11. Let us also apply some TLS rules so that the communication between the pods are encrypted

**enable-strict-mode.yml**

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
spec:
  mtls:
    mode: STRICT
```
Apply the enable-strict-mode configuration.
```
$ kubectl apply -f enable-strict-mode.yml
```

12. You can visualise these traffics with Kiali dashboard.

Apply the configurations 
```
$ kubectl apply -f istio-1.10.0/samples/addons/kiali.yaml
$ kubectl apply -f istio-1.10.0/samples/addons/prometheus.yaml
$ kubectl apply -f istio-1.10.0/samples/addons/extras/zipkin.yaml
$ kubectl apply -f istio-1.10.0/samples/addons/extras/prometheus-operator.yaml

```

![kiali.png](kiali.png?raw=true "kiali.png")

<br>
<hr>
<br>

# Wrapping Up
I hope this tutorial proivided you with a good high-level overview of Istio, how it works and how to leverage it for more sophisticated netwok routing. The implementation of scenarios that would otherwise requie a lot more time and resources. It is a powerful technology anyone looking into service mesh should consider.
