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

## A Multi-Node Service

Although we've tried something really simple and used the load balancer, we haven't yet used a multi-node service which is a more normal use for a load balancer.

Let's go ahead and install a new type of endpoint. We'll just put it in the default namespace for now and make have a number of replicas. Kubernetes will spread these replicas across the various worker nodes.

```shell
$ kubectl create deployment echoserver --image k8s.gcr.io/echoserver:1.3 --replicas 3
$ kubectl expose deployment echoserver --port=8080
```

We can see the three echoserver replicas running:

```
$ kubectl get pod -w
NAME                                  READY   STATUS    RESTARTS   AGE
echoserver-598b8497c-54bn7            1/1     Running   0          2m54s
echoserver-598b8497c-cvtxt            1/1     Running   0          2m54s
echoserver-598b8497c-dhlxz            1/1     Running   0          2m54s
```

Now we can do something new and update the previous ingress controller to also route traffic to the echoserver service.

Get the previous spec of the ingress using:

```shell
$ kubectl get ingress demo-localhost -o yaml > demo-localhost.yaml
```

Open the `demo-localhost.yaml` file and add a following new rule for the echoserver service to the spec rules:

```yaml
  - host: echoserver.localdev.me
    http:
      paths:
      - backend:
          service:
            name: echoserver
            port:
              number: 8080
        path: /
        pathType: Prefix
```

Check that the ingress controller still looks sane:

```shell
$ kubectl describe ingress

Name:             demo-localhost
Namespace:        default
Address:          192.168.1.254
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
Rules:
  Host                    Path  Backends
  ----                    ----  --------
  demo.localdev.me
                          /   demo:80 (10.244.2.5:80)
  echoserver.localdev.me
                          /   echoserver:8080 (10.244.1.4:8080,10.244.2.6:8080,10.244.3.4:8080)
Annotations:              <none>
Events:
  Type    Reason  Age                  From                      Message
  ----    ------  ----                 ----                      -------
  Normal  Sync    2m37s (x3 over 22h)  nginx-ingress-controller  Scheduled for sync
```

We have an ingress here where we're targetting all three echoserver instances on port `8080` within the cluster.

Let's go ahead and hit the load balancer.

The echoserver nginx config has a 301 redirect to https so we need to hit the loadbalancer on https to satisfy that. The nginx ingress controller will terminal TLS. It'll use a dummy TLS certificate for now as we have not configured a secret for the TLS certificate to exist in.

So curl to the load balancer to hit the echoserver service:

```shell
$ curl -vk --resolve echoserver.localdev.me:443:192.168.1.254 https://echoserver.localdev.me/
* Added echoserver.localdev.me:443:192.168.1.254 to DNS cache
* Hostname echoserver.localdev.me was found in DNS cache
*   Trying 192.168.1.254:443...
* Connected to echoserver.localdev.me (192.168.1.254) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/pki/tls/certs/ca-bundle.crt
*  CApath: none
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
* TLSv1.3 (IN), TLS handshake, Certificate (11):
* TLSv1.3 (IN), TLS handshake, CERT verify (15):
* TLSv1.3 (IN), TLS handshake, Finished (20):
* TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.3 (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / TLS_AES_256_GCM_SHA384
* ALPN, server accepted to use h2
* Server certificate:
*  subject: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  start date: Apr  4 22:35:10 2022 GMT
*  expire date: Apr  4 22:35:10 2023 GMT
*  issuer: O=Acme Co; CN=Kubernetes Ingress Controller Fake Certificate
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x55a355fecde0)
> GET / HTTP/2
> Host: echoserver.localdev.me
> user-agent: curl/7.79.1
> accept: */*
>
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* TLSv1.3 (IN), TLS handshake, Newsession Ticket (4):
* old SSL session ID is stale, removing
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 200
< date: Tue, 05 Apr 2022 21:39:21 GMT
< content-type: text/plain
< strict-transport-security: max-age=15724800; includeSubDomains
<
CLIENT VALUES:
client_address=10.244.2.3
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://echoserver.localdev.me:8080/

SERVER VALUES:
server_version=nginx: 1.9.11 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=echoserver.localdev.me
user-agent=curl/7.79.1
x-forwarded-for=10.244.2.1
x-forwarded-host=echoserver.localdev.me
x-forwarded-port=443
x-forwarded-proto=https
x-forwarded-scheme=https
x-real-ip=10.244.2.1
x-request-id=0c627ac72e9e14a28a90a7779bb9de41
x-scheme=https
BODY:
* Connection #0 to host echoserver.localdev.me left intact
-no body in request-
```

So, quite a lot going on in this echoserver request. It's much less noisy without the `-v` option by the way if you just want to see what the echoserver returns.

The requests actually get round-robined to the various nodes in the service. We can check the logs of all three echoserver replicas to see if they've seen any traffic. Do some more requests and then look at the logs for the contianers. First see what the containers are called:

```shell
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
echoserver-598b8497c-54bn7            1/1     Running   0          51m
echoserver-598b8497c-cvtxt            1/1     Running   0          51m
echoserver-598b8497c-dhlxz            1/1     Running   0          51m
...
```

So, we can see their logs in turn:

```shell
$ kubectl logs echoserver-598b8497c-54bn7
10.244.2.3 - - [05/Apr/2022:21:35:49 +0000] "GET / HTTP/1.1" 301 185 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:11 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:13 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:17 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
```

```shell
$ kubectl logs echoserver-598b8497c-cvtxt
10.244.2.3 - - [05/Apr/2022:21:39:09 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:12 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:14 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:20 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
```

```shell
$ kubectl logs echoserver-598b8497c-dhlxz
10.244.2.3 - - [05/Apr/2022:21:35:54 +0000] "GET / HTTP/1.1" 301 185 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:38:53 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:13 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:18 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
10.244.2.3 - - [05/Apr/2022:21:39:21 +0000] "GET / HTTP/1.1" 200 672 "-" "curl/7.79.1"
```

See that the times ripple across the containers in sequence as we call the service.

> **NOTE:** We haven't done any configuring on the ingress controller, but there are many annotations for the nginx ingress controller to set things like sticky sessions and what-have-you when you're ready to statr being serious about your configuration.

### A Sidetrack

If you want to get a little more involved you can alter the echoserver to return the hostname of the server that you reach. It makes a nice output to see in realtime which container you hit.

If you don't, move along to the next section...

Get a bash shell onto one of the echoservers. They're running nginx with a little lua script - which is funny because the echoserver example is used in the HAProxy ingress controller documentation.

>**NOTE:** Obviously replace the echoserver random characters with whatever matches your containers from the `get pod` command above.

For each of the echoserver nodes:

```shell
$ kubectl exec --stdin --tty echoserver-598b8497c-54bn7 -- /bin/bash
```

In `/etc/nginx/nginx.conf`, add the `server_hostname=` line below:

```shell
$ vi /etc/nginx/nginx.conf
```

```
    ngx.say("SERVER VALUES:")
    ngx.say("server_version=", "nginx: "..ngx.var.nginx_version.." - lua: "..ngx.config.ngx_lua_version)

```

Make sure you reload the config:

```shell
$ /usr/sbin/nginx -s reload
2022/04/05 22:08:59 [notice] 31#31: signal process started
```

End for!!

Now, everytime you reach an echoserver you will see exactly which node you hit:

```shell
$ curl -k --resolve echoserver.localdev.me:443:192.168.1.254 https://echoserver.localdev.me/
...

SERVER VALUES:
server_version=nginx: 1.9.11 - lua: 10001
server_hostname=echoserver-598b8497c-cvtxt
```

```shell
$ curl -k --resolve echoserver.localdev.me:443:192.168.1.254 https://echoserver.localdev.me/
...

SERVER VALUES:
server_version=nginx: 1.9.11 - lua: 10001
server_hostname=echoserver-598b8497c-54bn7
```
