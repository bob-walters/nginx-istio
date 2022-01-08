# nginx-istio

This is a project to demonstrate using Istio traffic management for A/B service shift with an nginx-ingress controller.

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

I wanted to run the nginx-ingress service under its own K8s namespace (`ingress`) to confirm that the traffic management from the nginx-ingress can work between namespaces.

The file `nginx-ingress-release.yaml` provides an nginx-ingress helm release that sets up a basic nginx-ingress service.  This can be installed with:

```
kubectl apply -f ingress-namespace.yaml

kubectl apply ingress-class.yaml

helm upgrade --install ingress-nginx-v4 ingress-nginx \
  -f nginx-ingress-release-values.yaml \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress --version="3.30.0"
```

Notes about the spec values used:

* The service type is `LoadBalancer`.  In Docker Desktop/Minicube environments, this will typically just expose the service via 'localhost'.  But the istio-ingressgateway is probably holding that role, so you may need to use a tunel to submit work to the nginx-ingress gateway if testing this in those environments (more later.)
* The release is deployed into the `"ingress"` namespace (separate from the namespace where the target services reside.  It doesn't have to be, but this demo shows that you can have a single nginx-ingress sending traffic to services in multiple/different namespaces)
* The Ingress class "nginx" is used to control the K8s Ingress definitions which will be served by this ingress controller.
* The Istio sidecar is injected into both the controller and the default backend based on the spec in the ingress namespace.

There is an annotations added to the nginx-ingres which is particularly important for this:

```
  podAnnotations:
    traffic.sidecar.istio.io/includeInboundPorts: ""
```

That ensures that traffic which is routed to that pod on ports 80 and 443 (i.e. the ingress ports it is meant to serve) do not go through the istio proxy and instead go directly to the controller container (nginx).  The outbound activity is meant to still go through the istio sidecar however.  This ensures that the Istio traffic management features won't ever side step the handling of the traffic by the nginx-ingress.


## Install podinfo-v1 and podinfo-v2 (sample services)

For the services that we are going to 
route traffic beteween, I'm just using two separately named
helm releases of the podinfo container to represent two versions of an application.  

We want istio sidecars injected for this set of services, just as we did for the nginx-ingress, so we start by setting the default namespace to automatically inject sidecars:

```
kubectl apply -f namespace-default.yaml
```

Note: you can use a dedicated namespace for these containers if you don't like the idea of using the default in this way, but note that it could be easier just to remove the added annotation after the experiments.

Installation of podinfo-v1 release:

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

Notice that the Ingress establishes routes for:

* `podinfo-v1.localhost.com` routes to podinfo-v1
* `podinfo-v2.localhost.com` routes to podinfo-v2

There are two important details about this ingress configuration:

* The annotation `nginx.ingress.kubernetes.io/service-upstream: "true"` is set in order to ensure that the nginx-ingress uses thw cluster IP address, rather than individual pod IP addresses, when forwarding traffic.

* The annotation `nginx.ingress.kubernetes.io/upstream-vhost: nginx-cache-v2.whitelabel-dev.svc.cluster.local` is **NOT** set here.  Setting this has the effect of altering the Host header to the value specified, and istio routes based on the Host header.  We use this later (below) with a separate Ingress resource to apply VirtualService routing for specific hosts where we want Virtual Services to apply.

To test the setup thus far, confirm at this point that you can see the ingress working and communicate with the podinfo container.  Start by checking whether the nginx-ingress service has an external-ip/port that you can access it at:

```
kubectl get services --all-namespaces

NAMESPACE      NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                      AGE
default        kubernetes                         ClusterIP      10.96.0.1        <none>        443/TCP                                      119d
ingress        ingress-nginx-v4-controller        LoadBalancer   10.105.82.242    <pending>     80:30022/TCP,443:31356/TCP                   30m
ingress        ingress-nginx-v4-default-backend   ClusterIP      10.105.29.49     <none>        80/TCP                                       30m
istio-system   istio-ingressgateway               LoadBalancer   10.106.133.137   localhost     15021:30119/TCP,80:31535/TCP,443:30168/TCP   3d19h
istio-system   istiod                             ClusterIP      10.101.116.7     <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP        3d19h
kube-system    kube-dns                           ClusterIP      10.96.0.10       <none>        53/UDP,53/TCP,9153/TCP                       119d
....
....
```

If you are running this on Docker Desktop, the external-ip of "localhost" will probably have been taken up by the istio-ingressgateway, and you'll need to do port forwarding to interact with the nginx-ingress.  The following will accomplish that:

```
kubectl port-forward --namespace=ingress service/ingress-nginx-v4-controller 8080:80
```

Then you can confirm that the ingress routing is working with the following curls:

```
# Should hit the default-podinfo service:
$ curl -H 'Host: podinfo-v1.localhost.com' localhost:8080

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
}
```

Note: Another way to accomplish the same is by using /etc/hosts to establish a fixed IP of 127.0.0.1 for the hostname "podinfo.localhost.com" and then you can actually `curl "http://podinfo.localhost.com:8080"` and see the correct routing.


## Adding the Ingress and Virtual Service for weighted routing

In order to use Istio's traffic management immediately after traffic is received by the nginx-ingress, you need to create a specific Ingress and Virtual Service configuration: 

* Create a K8s Service which represents the target of Ingress traffic which you want Istio to handle.  This K8s Service can be backed by any pod.
* Use a Separate K8s Ingress resource for each route that you want handled according to a specific Istio Virtual Service.  In that Ingress you will use the `nginx.ingress.kubernetes.io/upstream-vhost` annotation to specify the cluster.local hostname of the K8s service created in step 1.
* You'll define a Virtual Service (and any applicable Route Destinations) that applies for the cluster.local name of the service created in step 1.  The http routes you define in this Virtual Service will redirect any taffic that arrives for the Ingress you created in step 2.

This appears to be subject to some frailty at present (at least with Istio 1.12):

* From experimentation, both the Ingress and Virtual Service resources created MUST be in the same namespace.  I've used the namespace of the targetted services.  It does not need to be the namespace of the Ingress.  The routing described here did not work for me if they were in the 'ingress' namespace instead of the 'default' namespace.)
* The service's exposed port needs to be the same as the target port of the pods it routes to.  When I had the service `portinfo-svc-shift` serve port 8080 with target port 9898, and had the Ingress route to podinfo-svc-shift:8080, the virtual service did not apply.


In the example Virtual Service, I have use weighted routing in the Virutal Service definition to achieve the shift of traffic from podinfo-v1 to podinfo-v2 for the the FQDN `podinfo.localhost.com.`

For step 1, we create an additional service called podinfo-svc-shift which will initially just target podinfo-v1.

```
kubectl apply -f podinfo-svc-shift.yaml 
```

For step 2 - we can then create an Ingress resource which handles the FQDN podinfo.localhost.com, routing the traffic to the K8s service 'podinfo-svc-shift' that we just created

```
kubectl apply -f podinfo-ingress2-v1.yaml
```

If at this point, you issue calls to `curl -H 'Host: podinfo.localhost.com' localhost:8080` it just results in the traffic being routed to the podinfo-v1 pods per the K8s Service.  

The file `istio-virtualservice.yaml` contains the virtual service definition for the host "podinfo.localhost.com" which does weighted routing between the two versions of the podinfo service.

```
kubectl apply -f istio-virtualservice.yaml
```

### Pinning Sessions to a Version

If you inspect the virtual service definition you will see that it inspects a cookie to determine which service to route traffic to.  Upon first access (no cookie), the default route applies the weighted selection of one of the two versions and also adds a session cookie to the response.  This is done so that once a version is picked, the same version will apply for all requests made for that user's session.   without the session cookie, the traffic for subsequent HTTP requests would be routed according to the weight distribution each time.  This breaks any application relying on REST calls from the browser and even page-based or pre-rendered apps would have the user jumping between versions as they navigate.

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

There may be alternative ways to achieve this pinning effect, i.e. using session affinity, but this approach has one other benefit:  It is possible to deploy the next version of an application into a product environment, live, with with the weighting set to 100/0 so that all traffic is still going to the older/current version of the app.  In-house users can then test the new version of the application, in production, by deliberately setting a cookie value in their browser.  Once the new version has been validated, its deployment is just a matter of then adjusting the weights associated with the distribution.


## Alternative Approach that does not alter the Host header

One of the consequences of the approach just described is that the Host header on any inbound request will have been altered to reflect the value of the `nginx.ingress.kubernetes.io/upstream-vhost` annotation put on the Ingress definition.  This may be undesirable.  E.g. See https://github.com/kubernetes/ingress-nginx/issues/3171.

It is possible to avoid using that approach and leave the Host header unaltered, but the following changes are also required:

* The `hosts` of the Virtual Service is populated with the FQDNs being requested, rather than the K8s service that the Ingress is routing the traffic to.
* The Ingress must route the traffic to port 80 of some K8s service.  The service does not need to be exposed one port 80.  I.e. the Ingress can be invalid.

The fact that the routing inbound must be on port 80 s the subject of Issue https://github.com/istio/istio/issues/36705, as it should be possible to provide an explicit port in the criteria of the Virtual Service, but in my experience, that isn't working.  

When the host used in a virtual service definition does not coincide to any established K8s service, Istio defaults to applying the Virtual Service to traffic on port 80.  However even is an explicit port definiton is given in the Virtual Service, the routing did not appear to work, and using the debugging process detailed on https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/ (specifically, the inspection of the routes via `istioctl proxy-config routes {{source pod name}} --name {{target port}} -o json`) shows that the virtual service has not been put into effect for the configured port.

It may be possible that declaring a Destination that Istio understands would help alieviate this, but I have not experimented further with that.

An example configuration that routes in this way can be seen via the following files:

```
kubectl apply -f podinfo-ingress2-v2.yaml
```

The above alters the ingress definition to route podinfo.localhost.com to port 80 of the podinfo-svc-shift service.  This creates a broken/invalid Ingress definition.  However applying a change to the Virtual Service that is based on the requested hostname permits the weighted routing to work in spite of that:

```
kubectl apply -f istio-virtualservice-80.yaml
```

With this in place, you'll observe traffic routing correctly per the virtual service definition, but also 

### Details on Port Configurations Attempted:

I would appreviate any info/insights which might explain only port 80 works / what I am doing wrong / what I should be doing instead.

For example, here is the routing results that occur based on different configurations:

|Target Port of Ingress Rule|K8s Service Port|Vservice Port|Result|
|--------|--------|--------|----------|
|80|9898|-|works as desired|
|9898|9898|-|routes to K8s Service.  Virtual service has no effect|
|8080|9898|-|fails: timeout/502 while attempting to invoke service|
|9898|9898|9898|routes to K8s Service.  Virtual service has no effect|
|443|9898|-|fails: timeout/502 while attempting to invoke service|



