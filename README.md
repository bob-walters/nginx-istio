# nginx-istio
A project to demonstrate using Istio traffic management for A/B service shift with an nginx-ingress controller.

This is provided as a repeatable experiment.

Started with a stock K8s enviroment (e.g. docker-desktop) (Server Version 1.21.0)

## Install Istio

Install istio per [the instructions]([the instructions|https://istio.io/latest/docs/setup/install/istioctl/#generate-a-manifest-before-installation]):

```
./istio-1.12.1/bin/istioctl install
```

This should add a couple of deployments to the istio-system namespace:

```
$ kubectl get --all-namespaces pods
NAMESPACE      NAME                                     READY   STATUS    RESTARTS   AGE
istio-system   istio-ingressgateway-8c48d875-c9nxz      1/1     Running   0          102s
istio-system   istiod-58d79b7bff-jjg56                  1/1     Running   0          112s
....
```

Note: one consequence of installing istio is that it does declare a service of type "LoadBalancer", which if you are using Docker Desktop, MiniKube or similar, may mean that the istio-ingressgateway becomes bound to the external IP of 'localhost', and that the nginx-ingress service we intend to install (later) will not be directly accessible at http://localhost:80/.  However we can still successfully test the nginx-ingress service using port forwarding, as will be described later.


## Install nginx-ingress

I wanted to run the nginx-ingress service under its own K8s namespace to confirm that the routing can work between namespaces (as we use multiple in our production environments)

The file `nginx-ingress-release.yaml` provides an nginx-ingress helm release that sets up a basic nginx-ingress service.

```
kubectl apply -f ingress-namespace.yaml
kubectl apply ingress-class.yaml

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install -f nginx-ingress-release-values.yaml ingress-nginx-v4 ingress-nginx/ingress-nginx --version "3.30.0"
```

Or, if using the Helm Operator approach: 

```
kubectl apply -f nginx-ingress-release.yaml
```

Notes about the spec values:

* The service type is `LoadBalancer` based on the built-in support that Docker Desktop (and Minikube) have for that.  In those environments, this will typically just expose the service via 'localhost'.  But the istio-ingressgateway is probably holding that role, so you may need to use a tunel to submit work to the nginx-ingress gateway if testing this in those environments.
* The release is deployed into the "ingress" namespace (separate from the namespace where the target services reside.)
* The Ingress class "nginx" is used to control the K8s Ingress resources which will be used by this ingress controller
* The Istio sidecar is injected into both the controller and the default backend based on the spec in the ingress namespace.

There are a couple of annotations added to the nginx-ingres which are particularly important for this:

```
  podAnnotations:
    traffic.sidecar.istio.io/includeInboundPorts: ""
    traffic.sidecar.istio.io/excludeInboundPorts: "80,443"
    traffic.sidecar.istio.io/excludeOutboundPorts: "443"
```

The intention of the above is to ensure that traffic which is routed into that pod on ports 80 and 443 (i.e. the ingress ports it is meant to serve) do not go through the istio proxy and instead go directly to the controller container (nginx).  The outbound activity is meant to still go through the istio sidecar however.


## Install podinfo-v1 and podinfo-v2

For the services that we are going to 
route traffic beteween, I'm just going to use two separately named
instances of the podinfo container to represent two versions of an application.  

We want istio sidecars injected for this set of services, just as we did for the nginx-ingress, so we start by setting the default namespace to automatically inject sidecars:

```
kubectl apply -f namespace-default.yaml
```

Note: you can use a dedicated namespace for these containers if you don't like the idea of using the default in this way, but note that it could be easier just to remove the added annotation after the experiments.

Installation of podinfo-v1:

```
helm upgrade --install podinfo-v1 podinfo \
  -f podinfo-values.yaml \
  --repo https://stefanprodan.github.io/podinfo \
  --namespace default --version="3.2.0"
```

You should see the `podinfo-v1` service appear with an associated ClusterIP.  This service can't be reached externally, but we are going to reach it via the nginx-ingress.

Run the same process to create a separate deployment named podinfo-v2.

```
helm upgrade --install podinfo-v2 podinfo \
  -f podinfo-values.yaml \
  --repo https://stefanprodan.github.io/podinfo \
  --namespace default --version="3.2.0"
```


## Install an Ingress resource

```
kubectl apply -f podinfo-ingress-v1.yaml
```

That will establish an Ingress for both ofthe previous podinfo releases where the routing is determined by the hostname (host header) used in Http requests.

There are two important details about this ingress configuration:

* The annotation `nginx.ingress.kubernetes.io/service-upstream: "true"` is set in order to ensure that the nginx-ingress uses thw cluster IP address, rather than individual pod IP addresses, when forwarding traffic.

* The annotation `nginx.ingress.kubernetes.io/upstream-vhost: nginx-cache-v2.whitelabel-dev.svc.cluster.local` is **NOT** set.  Many articles will indicate that you should typically set this to coincide with the above, but setting this has the effect of altering the Host header to the value specified, and istio routes based on the Host header, so setting this would require that all Istio routing rules be specified in terms of those hostnames and not the original hostnames.  More details on this can be found at: https://github.com/kubernetes/ingress-nginx/issues/3171

Confirm at this point that you can see the ingress working and communicate with the podinfo container.  Start by checking whether the nginx-ingress service has an external-ip/port that you can access it at:

```
kubectl get services --all-namespaces

NAMESPACE      NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
default        default-podinfo                    ClusterIP      10.96.216.5      <none>        9898/TCP,9999/TCP                            3d16h
default        kubernetes                         ClusterIP      10.96.0.1        <none>        443/TCP                                      119d
ingress        ingress-nginx-v4-controller        LoadBalancer   10.105.82.242    <pending>     80:30022/TCP,443:31356/TCP                   30m
ingress        ingress-nginx-v4-default-backend   ClusterIP      10.105.29.49     <none>        80/TCP                                       30m
istio-system   istio-ingressgateway               LoadBalancer   10.106.133.137   localhost     15021:30119/TCP,80:31535/TCP,443:30168/TCP   3d19h
istio-system   istiod                             ClusterIP      10.101.116.7     <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        3d19h
kube-system    kube-dns                           ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP                       119d
```

If you are running this on Docker Desktop, the external-ip of "localhost" will already have been taken up by the istio-ingressgateway, and you'll need to do port forwarding to interact with the nginx-ingress.  The following will accomplish that:

```
kubectl port-forward --namespace=ingress service/ingress-nginx-v4-controller 8080:80
```

Then you can confirm that the ingress routing is working with the following curls:

```
# Should hit the default backend since there is no routing defined for 'localhost'
$ curl localhost:8080
default backend - 404
```

To simulate requests being made against our ingress hostname, we can explicitly set the host header and should then see the correct routing.

```
# Should hit the default-podinfo service:
$ curl -H 'Host: podinfo.localhost.com' localhost:8080
{
  "hostname": "default-podinfo-868fdf8cb5-m5zqw",
  "version": "3.2.0",
  "revision": "7a8b7d6a5c27725f81a94594a1de04a749908df2",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v3.2.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.13.1",
  "num_goroutine": "8",
  "num_cpu": "4"
}
```

Note: Another way to accomplish the same is by using /etc/hosts to establish a fixed IP of 127.0.0.1 for the hostname "podinfo.localhost.com" and then you can actually curl "http://podinfo.localhost.com:8080" and see the correct routing.  

## Adding the Virtual Service to do weighted routing

The file `istio-virtualservice.yaml` contains the virtual service definition for the hosts "podinfo.localhost.com" (deliberately separate).

```
kubectl apply -f istio-virtualservice.yaml
```

One applied, attempting to hit `curl -H 'Host: podinfo.localhost.com' localhost:8080` just results in the default-backend being hit.  We do need to have an Ingress route defined for podinfo.localhost.com as otherwise the nginx controller is routing it to the backend.  We can add a rule for that hostname by applying: 

```
kubectl apply -f  podinfo-ingress-v2.yaml
```

Doing that makes the routing based on that hostname work, but it also routes entirely according to the ingress rule, and the virtual service does not apply.

Ultimately, to get the istio routing to take effect, we have to leverage an important aspect of both the K8s nginx-ingress behavior, and istio itself.  We configure the target port of 80. We change that routing rule to:

```
  - host: "podinfo.localhost.com"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: podinfo-v2
            port:
              number: 80
``` 

If we investigate the ingress defintion, we can see that K8s reports this as invalid, but it still ends up working.  Here's what I think is happening:

* the nginx-ingress service will, "try" to route traffic to port 80 of the IP address that it can determine based on the ClusterIP, even though the destination appears invalid.
* At that point, the traffic is routed from the nginx-controller to its sidecar as "outbound" traffic.  (I.e. per the standard way that the istio mesh works)  The sidecar will note that the traffic corresponds to the Host 'podinfo.localhost.com' and port 80, and based on that will apply the VirtualService routing rule based on that matching criteria, which then has the effect of changing the destination and applying the Virtual Service as expected.



You can see that a request sent to podinfo.localhost.com is assigned randomly (50/50) to one of the two sites, and that a "set-cookie" header is returned to pin the session to that version of the app.

```
$ curl -v -H 'Host: podinfo.localhost.com' localhost:8080
*   Trying ::1:8080...
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: podinfo.localhost.com
> User-Agent: curl/7.77.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 27 Dec 2021 18:51:43 GMT
< Content-Type: application/json; charset=utf-8
< Content-Length: 395
< Connection: keep-alive
< x-content-type-options: nosniff
< x-envoy-upstream-service-time: 2
< set-cookie: my-site-version=1; HttpOnly; Path=/
< X-Request-ID: 889dba4ae49e2a9cb2c79a6c6e79c8cd
< 
{
  "hostname": "podinfo-v1-6b8898549c-qdr75",
  "version": "3.2.0",
  "revision": "7a8b7d6a5c27725f81a94594a1de04a749908df2",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v3.2.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.13.1",
  "num_goroutine": "8",
  "num_cpu": "4"
* Connection #0 to host localhost left intact

```

In all subsequent requests, the session cookie has the effect of ensuring that that user is returned to the same version of the app for the duration of their session.

```
$ curl -v -H 'Host: podinfo.localhost.com'-H "Cookie: my-site-version=2" localhost:8080
*   Trying ::1:8080...
* Connected to localhost (::1) port 8080 (#0)
> GET / HTTP/1.1
> Host: podinfo.localhost.com
> User-Agent: curl/7.77.0
> Accept: */*
> Cookie: my-site-version=2
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 27 Dec 2021 19:03:16 GMT
< Content-Type: application/json; charset=utf-8
< Content-Length: 394
< Connection: keep-alive
< x-content-type-options: nosniff
< x-envoy-upstream-service-time: 1
< X-Request-ID: 2144a6e5f3abea686f63955bfe62f820
< 
{
  "hostname": "podinfo-v2-f5cb48c4d-tbtbk",
  "version": "3.2.0",
  "revision": "7a8b7d6a5c27725f81a94594a1de04a749908df2",
  "color": "#34577c",
  "logo": "https://raw.githubusercontent.com/stefanprodan/podinfo/gh-pages/cuddle_clap.gif",
  "message": "greetings from podinfo v3.2.0",
  "goos": "linux",
  "goarch": "amd64",
  "runtime": "go1.13.1",
  "num_goroutine": "8",
  "num_cpu": "4"
* Connection #0 to host localhost left intact
}
```



# Any better ways?

I would appreviate any info/insights which might explain why this works / what I am doing wrong / what I should be doing instead.

Some of the above is substantiated by the Istio troubleshoting at https://istio.io/latest/docs/ops/common-problems/network-issues/#route-rules-have-no-effect-on-ingress-gateway-requests.  Best I can determine - if the Ingress rule targets a valid service:port that K8s can handle by itself, then istio doesn't appear to take effect, or the Istio VirtualService needs specific port matching (which I have tried, unsuccessfully) to get the routing working.

For example, here is the routing results that occur based on different 

|Target Port of Ingress Rule|K8s Service Port|Vservice Port|Result|
|--------|--------|--------|----------|
|80|9898|-|works as desired|
|9898|9898|-|routes to K8s Service.  Virtual service has no effect|
|8080|9898|-|fails: timeout/502 while attempting to invoke service|
|9898|9898|9898|routes to K8s Service.  Virtual service has no effect|
|443|9898|-|fails: timeout/502 while attempting to invoke service|


I have used my forehead to put a sizeable dent into several walls trying to find a more 'intuitive' (less hacky) configuration for this, and have had no luck.  For the record, my experiments have tried:

* Basing the VirtualService on the internal cluster hostname that ingress is routing to.
* Using an explicit port value in the rules of the VirtualService (trying to get it to work with something other than port 80)
* Creating explicit services not backed by any pods which are then the target of the Ingress routing (with the expectation that it would hand-off to Istio in the process.)
* Adding port 9898 explicily to the `traffic.sidecar.istio.io/includeOutboundPorts` annotation on the nginx-ingress pod.

What I've observed:

* In general, if the ingress routing definition is accurate (i.e. targets a valid service:port combination), then istio doesn't seem to take effect, at least from the point of nginx-ingress to the K8s services.  I have virtual service rules work fine however for service to service routing, this odity only seems to apply to traffic arriving at the nginx-ingress.


