## Ingress and Loadbalancing

Now that the cluster is up (Back to [pt1](1-installation.md) if that's not the case) we can start getting down to business and configuring things. In this section we will create a load balancer and a service to use the load balancing.

The load balancer we'll use is from the awesome [metallb](https://metallb.universe.tf/) project which aims to solve our problem - no load balancing on a plain k8s cluster.

### Why Do We Need Loadbalancing?

When we start creating services in our cluster we need to be able to access those services. When we do, we need a way of exposing the service so we can access it and route traffic to it.

There are options such as tying a service to a NodePort which exposes the service on a worker node IP address and port. The problem with this is then that we'd need to have an external load balancer to balance the traffic across the different nodes with the relevant health check, etc.

The alternative is, to me, neater - as Kubernetes has all the knowledge of the services, it can create the network routing necessary to keep a service alive as containers are brought down and up on different nodes. Perhaps when we're patching a worker node for example and we have to shift all of the containers off that node first in order to be able to patch it.

### Installation of MetalLB

Installation is really straight forward. We'll use helm (I really like helm) to install with a little configuration. All you're going to need is another IP address that cannot be given out by your DHCP service and is essentially reserved for this load balancer. You can choose a range of IP addresses for metalLB to use for load balancing if you have some more available and think you'll have enough services.

We need a configuration file because we need to be able to tell metalLB what IP addresses it might give out:

```
$ cat config/metallb-config.yaml
configInline:
  address-pools:
   - name: default
     protocol: layer2
     addresses:
     - 192.168.1.254/32
```

With the config file we can now go ahead and install the load balancer in the cluster:

:::note

These shell snippets assume you still have the `KUBECONFIG` environment variable set and pointing to the `kubeconfig` file we downloaded earlier.

:::

```shell
$ helm repo add metallb https://metallb.github.io/metallb
```

```shell
$ helm install metallb metallb/metallb -f config/metallb-config.yaml
NAME: metallb
LAST DEPLOYED: Mon Apr  4 23:27:28 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
MetalLB is now running in the cluster.
LoadBalancer Services in your cluster are now available on the IPs you
defined in MetalLB's configuration:

config:
  address-pools:
  - addresses:
    - 192.168.1.254/32
    name: default
    protocol: layer2

To see IP assignments, try `kubectl get services`.
```

Now we have an IP address that we can use to put a service on. Woohoo! But before we get ahead of ourselves we next need an ingress controller that will make use of the load balancer.

## Nginx Ingress Controller

The [nginx ingress controller project](https://kubernetes.github.io/ingress-nginx/deploy/) is coming to our rescue to provide us with an ingress controller we can use to route traffic to a service.

Again, we'll use `helm` to install the ingress controller.

Installing is really a one-liner:

```
$ helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

This produces, well, a load of gumpf on the screen. But it's worth a quick read. The synopsis here is to run the following output:

```shell
$ kubectl --namespace ingress-nginx get services -o wide -w ingress-nginx-controller
NAME        ingress-nginx-controller
TYPE        LoadBalancer
CLUSTER-IP  10.101.132.87
EXTERNAL-IP 192.168.1.254
PORT(S)     80:31340/TCP,443:32255/TCP
AGE         89s
SELECTOR    app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```

I've reformatted the output so it's a bit easier to see in this page. However, the most important thing to note here is that the `EXTERNAL-IP` is per the configuration we passed in and the load balancer has been configured on an IP address separate to the cluster node IPs.

We can go ahead and ping the service and see a response at least:

```shell
$ curl -v http://192.168.1.254/
*   Trying 192.168.1.254:80...
* Connected to 192.168.1.254 (192.168.1.254) port 80 (#0)
> GET / HTTP/1.1
> Host: 192.168.1.254
> User-Agent: curl/7.79.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 404 Not Found
< Date: Mon, 04 Apr 2022 22:41:22 GMT
< Content-Type: text/html
< Content-Length: 146
< Connection: keep-alive
<
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
* Connection #0 to host 192.168.1.254 left intact
```

## Create a Loadbalanced Service

To test the load balancer we'll create our own load balanced service:

```shell
$ kubectl create deployment demo --image=httpd --port=80
$ kubectl expose deployment demo
$ kubectl create ingress demo-localhost --class=nginx --rule='demo.localdev.me/*=demo:80'
```

We have created a rule in the ingress controller to say that the host `demo.localdev.me` should be routed to the service demo on port 80. Don't forget you can use `kubectl get services --all-namespaces` to see what services are currently running!

We can now use the LoadBalancer to get to our demo service and get the "It works!" nginx holding page back. Not terribly exciting, but you know - not a bad result either.

:::note

You can of course modify your hosts file as well if you want, but that generally requires sudo and you can tell modern versions of curl to resolve any hostname and port to any IP address. It's easier to do a one-liner most of the time unless you need regular access through a browser or something

:::

```shell
$ curl -v --resolve demo.localdev.me:80:192.168.1.254 http://demo.localdev.me/
* Added demo.localdev.me:80:192.168.1.254 to DNS cache
* Hostname demo.localdev.me was found in DNS cache
*   Trying 192.168.1.254:80...
* Connected to demo.localdev.me (192.168.1.254) port 80 (#0)
> GET / HTTP/1.1
> Host: demo.localdev.me
> User-Agent: curl/7.79.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 04 Apr 2022 22:50:29 GMT
< Content-Type: text/html
< Content-Length: 45
< Connection: keep-alive
< Last-Modified: Mon, 11 Jun 2007 18:53:14 GMT
< ETag: "2d-432a5e4a73a80"
< Accept-Ranges: bytes
<
<html><body><h1>It works!</h1></body></html>
* Connection #0 to host demo.localdev.me left intact
```
