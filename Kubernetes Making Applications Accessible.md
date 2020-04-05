# Kubernetes: Making Applications Accessible
Agenda
* [Understanding Pod Access Options](bear://x-callback-url/open-note?id=38358CEF-9AB5-4FA8-A605-89EDFF565505-29447-0009B7716F1964E3&header=Understanding%20Pod%20Access%20Options)
* [Exposing Pods with Services](bear://x-callback-url/open-note?id=38358CEF-9AB5-4FA8-A605-89EDFF565505-29447-0009B7716F1964E3&header=Exposing%20Pods%20with%20Services)
* [Understanding Services and DNS](bear://x-callback-url/open-note?id=38358CEF-9AB5-4FA8-A605-89EDFF565505-29447-0009B7716F1964E3&header=Understanding%20Services%20and%20DNS)
* [Using Ingress](bear://x-callback-url/open-note?id=38358CEF-9AB5-4FA8-A605-89EDFF565505-29447-0009B7716F1964E3&header=Using%20Ingress)
* Understanding Kubernetes Networking


[LucidChart Diagram](https://www.lucidchart.com/documents/edit/a5572f21-e0fe-4ad3-a810-50b6795644ab)
[[Running K3s with Multipass on Macbook]]
Previous Session: [[Kubernetes]]

## Understanding Pod Access Options
* Service Object
	* Types
		* `ClusterIP` - useless; internal only
		* `nodePort` - binds a port to all nodes; 
			* all you got for minikube/multipass
		* `LoadBalancer` - 
* Ingress
	* load balances between different services
	* url (L7) load balancing


## Exposing Pods with Services
create service object
show different types
change service type

### Access Deployment from within the cluster using Service Object

create service object

```
alias k=kubectl

# Create a deployment
kubectl create deployment --image=nginx nginx
# 
kubectl get deployments
# See running services
kubectl get svc
# Create service for nginx deployment; point to port 80 on node
kubectl expose deployment nginx --port=80

$ k get svc
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
...
nginx          ClusterIP   10.43.86.254   <none>        80/TCP           38s

$ k describe svc nginx
Name:              nginx
Namespace:         default
Labels:            app=nginx
Annotations:       <none>
Selector:          app=nginx
Type:              ClusterIP
IP:                10.43.86.254
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.42.2.19:80
Session Affinity:  None
Events:            <none>

# NAT 80 to 80; notice the endpoints; ; only one

# scale out deployment
k scale --replicas=3 deployment nginx
k get pods -w   #watch for all pods to be Running

$ k describe svc nginx | egrep "IP|Endpoints"
Type:              ClusterIP
IP:                10.43.86.254
Endpoints:         10.42.0.11:80,10.42.2.19:80,10.42.2.20:80

# connect to hypervisor to test
multipass connect k3s-master
# or
minikube ssh

# connect to ClusterIP from host
curl 10.43.86.254
## should get nginx output
```


### Access the Deployment from outside cluster, expose it as a NodePort type

Exit out of hypervisor
Change Service type to NodePort
```
#kubectl expose deployment hello-minikube --type=NodePort

# Change to NodePort; specific port to NAT
k edit svc nginx
## Change "type: ClusterIP" to "type: NodePort"
## Add "nodePort: 31375" below "targetPort"

# Confirm service setup 
$ k get svc nginx
NAME    TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.43.86.254   <none>        80:32000/TCP   21m

#------------------------------
# Now test from outside cluster
#------------------------------
multipass list

```



# Understanding Services and DNS
```
# Confirm confirm coredns pod is running
multipass exec k3s-master -- sudo k3s crictl ps | grep dns

# List services
$ k get svc
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
...
nginx          NodePort    10.43.86.254   <none>        80:32000/TCP     4h4m

# List endpoints for service "nginx"
k get endpoints nginx
NAME    ENDPOINTS                                               AGE
nginx   10.42.0.11:80,10.42.0.12:80,10.42.0.13:80 + 3 more...   4h5m

```

```
$ cat busybox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busydns
  namespace: default
spec:
  containers:
  - image: busybox
    name: busy
    command:
      - sleep
      - "300"
```


```
$ k apply -f busybox.yaml

$ k exec -it busydns -- nslookup nginx
Server:		10.43.0.10
Address:	10.43.0.10:53

Name:	nginx.default.svc.cluster.local
Address: 10.43.86.254
...
```

The address for `nginx.default.svc.cluster.local` is the ClusterIP for the `nginx` service.

### Limitation of the Service API
* Manual configuration of load balancer when using NodePort type
* Potential latency due to the network hops introduced by kube-proxy
* A load balancer per service (cloud) can quickly escalate operational costs
* The Service API cannot cater for advanced ingress traffic patterns (L4 only)

- - - -


## Using Ingress

* Module Outline
	* Introduce the Ingress concept and API
	* Discuss the nature of ingress controllers
	* Differentiate between host-based and path-based routing
	* Learn hot to define ingress objects to route requests to backends

* **Ingress** - Kubernetes API object that manages the routing of external HTTP/S traffic to services running in a cluster
	* main difference vs NodePort or LoadBalancer types is focus on L7 HTTP/S traffic
* **Ingress Object Definition**
* **Ingress Controller** - need to install separately
* Why Third-party Controllers?
	* part of the controllers's function is to act as a reverse proxy for cluster workloads
	* ability to choose preferred best of breed solution
	* infrastructure providers can create ingress controllers optimized for their environments
		* AWS and GCP have controllers specific to their platform
* Proxy-based Ingress Controllers
	* **Nginx** - maintained by the k8s community; https://git.io/fh4UC
	* **Traefik** - based on a cloud native edge router
		* 🔥 https://docs.traefik.io/v1.7/user-guide/kubernetes/
		* https://docs.traefik.io/user-guides/crd-acme/
	* **Contour** - Uses the popular Envoy service proxy; https://git.io/fh4Ua
* Characteristics of Ingress API
	* Defines traffic routes between external clients and services
	* Allows encrypted communication using TLS
	* Load balances traffic across a services's endpoints (pods)
	* Still beta; is likely to evolve over time
* Name-based Virtual Hosts
	* 	Single IP associated with multiple host names
	* Controller strips host name header and sends to relevant backend service
* Path-based Routing
	* Route to backend service based on URL path
	* Controller strips prefix path and sends to relevant backend service
		* ex
			* www.cheese.com/cheddar - strips cheddar and send to service A
			* www.cheese.com/manchego - strips manchego and send to service B

### Deploy the Traefik Ingress Controller

#### Deploy the open source Traefik ingress controller

If minikube, need to disable default Nginx controller: `minikube addons disable ingress`

```
k get no
k get clusterrole
k describe clusterrole traefik
k get clusterrole traefik -o yaml
```

```
$ k get daemonset -A
NAMESPACE    NAME           DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  ...  AGE
kube-system  svclb-traefik  3        3        3      3           3          ...  3d23h
```

```
# From specific namespace, get all objects matching label.
k -n kube-system get -l app.kubernetes.io/name=traefik-ingress all
```

### Providing a Default Backend Service

```
spec:
# define default backend service
  backend:
    serviceName: default-backend-svc
    servicePort: 8080
```

* Limitations of the Default Backend Service
	* 	Perfect design for routing ingress traffic to a single backend service
	* Works with ingress definitions containing rules for HTTP requests
	* Not so good when there are multiple default backend definitions
		* Ingress controllers use different mechanisms for resolving this issue

### Define Ingress Routing Rules

#### Host Rule Routing

```
spec:
  rules: 
  #ingress host rule requires host field and list of HTTP paths
  - host: www.dibble.sh 
    http:
      paths:
        backend:
          serviceName: foo-svc
          servicePort: 8080
```

#### Path Rule Routing
* Route to different backend services based on path in URL
	* www.dibble.sh/bar --> bar-svc:8080
	* www.dibble.sh/foo --> foo-svc:8080

```
spec:
  rules: 
  - http:
      paths:
      - path: /foo #path field specifies the required path of the URL request
        backend:
          serviceName: foo-svc
          servicePort: 8080
      - path: /bar #additional paths can be added to the list
        backend:
          serviceName: bar-svc
          servicePort: 8080
```

* **Path Rule Considerations**
	* **Regex** - paths are expressed as extended POSIX regex
	* **Rewrite** - request paths may need to be rewritten
		* This is specified in annotations
	* **Priority** - may be necessary to prioritize paths

## Demo
* Define and deploy an ingress for default backend service
* Create an ingress object for host-based routing
* Reconfigure the ingress object for path-based routing


1. Create host entries for IP of multipass VM
```
echo "$(multipass list | grep master | awk '{print $3}') \
ingress.example.com red.ingress.example.com blue.ingress.example.com" | \
sudo tee -a /etc/hosts
```


Create deployments and services.
```
for i in nginxhello-*.yaml; do kubectl apply -f $i; done
```

Sample output
```
deployment.apps/nginxhello-blue created
service/nginxhello-blue created
deployment.apps/nginxhello-default created
service/nginxhello-default created
deployment.apps/nginxhello-red created
service/nginxhello-red created
```

We are only using ClusterIP (vs NodePort) so the services/pods are not externally accessible. Need an ingress.

```
cd ../ingress-scenarios/
```

```
# First ingress definition is very simple
$ cat single-default-backend-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: single-default-backend-ingress
spec:
  backend:
    serviceName: nginxhello-default
    servicePort: 80
```

Create ingress
```
k apply -f single-default-backend-ingress.yaml
```

Describe ingress single-default-backend-ingress
```
$ k describe ing single-default-backend-ingress
Name:             single-default-backend-ingress
Namespace:        default
Address:          192.168.64.9
Default backend:  nginxhello-default:80 (10.42.3.6:80)
Rules:
  Host        Path     Backends
  ----        ----     --------
  *           *        nginxhello-default:80 (10.42.3.6:80)
Annotations:  Events:  <none>
```

open ingress.example.com

You will see
![](Kubernetes%20Making%20Applications%20Accessible/FC8FF3F9-5AF0-4978-BE1C-290FBA107D1B.png)

* Delete ingress with simple rules
```
k delete ing single-default-backend-ingress
```

* Examine `host-rule-ingress.yaml`.
```
$ cat host-rule-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: host-rule-ingress
spec:
  backend:
    serviceName: nginxhello-default
    servicePort: 80
  rules:
  - host: blue.ingress.example.com
    http:
      paths:
      - backend:
          serviceName: nginxhello-blue
          servicePort: 80
  - host: red.ingress.example.com
    http:
      paths:
      - backend:
          serviceName: nginxhello-red
          servicePort: 80

```

* Create ingress with host-base rules.
```shell
$ k apply -f host-rule-ingress.yaml
ingress.extensions/host-rule-ingress created
```

* Describe ingress.
```
$ k describe ing host-rule-ingress
Name:             host-rule-ingress
Namespace:        default
Address:          192.168.64.9
Default backend:  nginxhello-default:80 (10.42.3.6:80)
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  ingress.example.com
                                nginxhello-blue:80 (10.42.3.5:80)
Annotations:           Events:  <none>
```

### Test ingress with host rules.

* Browse to http://ingress.example.com. You will see a gray container. 
	* Default backend because neither rule matched.

* Browse to http://blue.ingress.example.com. You will see a blue container.
	* Matched rule for blue host.
![](Kubernetes%20Making%20Applications%20Accessible/9F0390D7-95A8-482E-BEC0-4AD9D06E0FF5.png)

* Browse to http://red.ingress.example.com. You will see a red container.
	* 	Matched rule for red host.

* Delete ingress
```
k delete ing host-rule-ingress
```

### Path Rule

* Examine `path-rule-ingress.yaml`.
```
$ cat path-rule-ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: path-rule-ingress
  annotations:
    traefik.frontend.rule.type: PathPrefixStrip #Tells controller to strip prefix
spec:
  backend:
    serviceName: nginxhello-default
    servicePort: 80
  rules:
  - http:
      paths:
      - backend:
          serviceName: nginxhello-blue
          servicePort: 80
        path: /blue
  - http:
      paths:
      - backend:
          serviceName: nginxhello-red
          servicePort: 80
        path: /red
```
	* ingress with path rules defined for `<fqdn>/red` and `<fqdn>/blue`. 
	* Note, the ingress controller annotation
```
  annotations:
    traefik.frontend.rule.type:
```

17. Create the ingress object for the path-based rule.

```
kubectl apply -f path-rule-ingress.yaml
```

18. Inspect the created object, and confirm it's as you expect it to be.

```
kubectl describe ing path-rule-ingress
```

19. Use a web browser to navigate to ingress.example.com/red and ingress.example.com/blue, respectively, and confirm that the ingress controller directs you to the appropriate backend. Try ingress.example.com, too.
 

This Ingress config is for Traefik ingress controller.
```shell
tee ingress.yaml <<EOF
#apiVersion: networking.k8s.io/v1beta1
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: single-default-backend-ingress # test-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
    traefik.frontend.rule.type: PathPrefixStrip
spec:
# define default backend service
  backend:
    serviceName: default-backend-svc
    servicePort: 8080
  rules:
  - host: ingress.example.com
    http:
      paths:
      - path: /testpath
        backend:
          serviceName: nginx
          servicePort: 80
EOF
```
* `kubernetes.io/ingress.class: "traefik"`
* `traefik.frontend.rule.type: PathPrefixStrip` - Strips the path

* Need to install ingress controller
	* Traefik installed by default
```

```


Apply
```
kubectl apply -f ingress.yaml
```

#### Inspect the API objects created in the cluster


```
kubectl get ingress
```

Sample Output
```
NAME           HOSTS                 ADDRESS        PORTS   AGE
test-ingress   ingress.example.com   192.168.64.9   80      12s
```

```
curl http://ingress.example.com/testpath
```

- - - -

* Output
`service/hello-minikube exposed`
* Check if the Pod is up and running
	* `kubectl get pod`
	* If the output shows the **STATUS** as **ContainerCreating**, the Pod is still being created:
```
NAME                              READY     STATUS              RESTARTS   AGE
hello-minikube-3383150820-vctvh   0/1       ContainerCreating   0          3s
```

	* If the output shows the **STATUS** as **Running**, the Pod is now up and running:
```
NAME                              READY     STATUS    RESTARTS   AGE
hello-minikube-3383150820-vctvh   1/1       Running   0          13s
```

* Get the URL of the exposed Service to view the Service details
```
minikube service hello-minikube --url
```
	* Output
`http://192.168.99.101:31050`
* Browse to the URL. Or...
* Launch a browser to the URL using `minikube`.
```
minikube service hello-minikube
```

	* Output
```
Hostname: hello-minikube-856979d68c-pjrd2

Pod Information:
    -no pod information available-

Server values:
    server_version=nginx: 1.13.3 - lua: 10008

Request Information:
    client_address=172.17.0.1
    method=GET
    real path=/
    query=
    request_version=1.1
    request_scheme=http
    request_uri=http://192.168.99.101:8080/

Request Headers:
    accept=text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
    accept-encoding=gzip, deflate
    accept-language=en-US,en;q=0.5
    connection=keep-alive
    host=192.168.99.101:31050
    upgrade-insecure-requests=1
    user-agent=Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:68.0) Gecko/20100101 Firefox/68.0

Request Body:
    -no body in request-
```

* View the **Deployment**
```
$ kubectl get deployments
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
hello-minikube   1/1     1            1           59s
```

		* View the Services
```
$ kubectl get services
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
hello-minikube   NodePort    10.97.202.64   <none>        8080:30007/TCP   60s
kubernetes       ClusterIP   10.96.0.1      <none>        443/TCP          34m
```

* View cluster events
	* `kubectl get events`
* View kubectl configuration
```
	* kubectl config view
```
* View logs
	* Get pod info
```
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-84d6886696-nqk62   1/1     Running   0          17m
```

	* Get logs for pod
```
$ kubectl logs hello-node-84d6886696-nqk62
Received request for URL: /
Received request for URL: /favicon.ico
```

* See in dashboard
```
minikube dashboard
```
* Delete the `hello-minikube` Service.
```
kubectl delete services hello-minikube
```
	* Output
	* `service "hello-minikube" deleted`
* Delete the `hello-minikube` Deployment
```
kubectl delete deployment hello-minikube
```
	* Output
	* `deployment.extensions "hello-minikube" deleted`
