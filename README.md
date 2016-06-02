# junio


<!--
- Notes:

- Kubernetes Cluster:

  - Docker:
    - "an application container technology."

  - Etcd:
    - "a distributed key-value datastore that manages cluster-wide information
       and provides service discovery.

  - Flannel:
    - "an overlay network fabric enabling container connectivity across
       multiple servers."


  - etcd.

  - flanneld - overlay network.
    - Stores configuration in etcd.

  - Comprises a master and a set of nodes.

  - Running on master:

    - kube-apiserver
      - "REST API that validates and configures data for API objects such as
         pods, services, replication controllers."

    - kube-scheduler
      - "A policy-rich, topology-aware, workload-specific function that
         significantly impacts availability, performance, and capacity."

  - Running on nodes:

    - kube-proxy
      - "Can do simple TCP/UDP stream forwarding or round-robin TCP/UDP
         forwarding across a set of backends."

    - kubelet
      - "The primary 'node agent' that runs on each node.  The kubelet takes
         a set of PodSpecs and ensures that the described containers are running
         and healthy."


- Concepts:

  - Deployment
    "Deployment objects automate deploying and rolling updating applications.
     Compared with kubectl rolling-update, Deployment API is much faster, is
     declarative, is implemented server-side and has more features (for example,
     you can rollback to any previous revision even after the rolling update
     is done)."

  - Job
    "A job creates one or more pods and ensures that a specified number of
     them successfully terminate.  As pods successfully complete, the job
     tracks the successful completions.  When a specified number of successful
     completions is reached, the job itself is complete.  Deleting a Job will
     cleanup the pods it created."

  - Volume
    "On-disk files in a container are ephemeral, which presents some problems
     for non-trivial applications when running in containers.  First, when a
     container crashes kubelet will restart it, but the files will be lost -
     the container starts with a clean slate.  Second, when running containers
     together in a Pod it is often necessary to share files between those
     containers.  The Kubernetes Volume abstraction solves both of these
     problems."

  - PodIP

  - ClusterIP


- Concepts:

  - Pod:
    - Group of one or more Docker containers running on the same host.

  - Replication Controller:
    - Manage the lifecycle of pods.
    - The controller maintains a desired number of pod replicas and will
      automatically create or kill pods as necessary.

  - Services:
    - Collections of pods that are exposed with a single and stable name and
      network address.
    - The service provides load balancing to the underlying pods,
      with or without an external load balancer.

  - Label
    - A simple name-value pair.

  - Hyperkube:
    - All-in-one binary running Kubernetes cluster.

  - PodIP

  - ClusterIP


- Networking:

  "Valid values for the ServiceType field are:"

   ClusterIP
   - Default.
   - Reachable from inside the cluster only.

   NodePort
   - "On top of having a cluster-internal IP, expose the service on a port on
      each node of the cluster (the same port on each node)."
   - "You’ll be able to contact the service on any <NodeIP>:NodePort address."

   LoadBalancer
   - "On top of having a cluster-internal IP and exposing service on a NodePort
      also, ask the cloud provider for a load balancer which forwards to the
      Service exposed as a <NodeIP>:NodePort for each Node."

   "Unlike Pod IP addresses, which actually route to a fixed destination,
    Service IPs are not actually answered by a single host.  Instead, we use
    iptables (packet processing logic in Linux) to define virtual IP addresses
    which are transparently redirected as needed.  When clients connect to the
    VIP, their traffic is automatically transported to an appropriate endpoint.
    The environment variables and DNS for Services are actually populated in
    terms of the Service’s VIP and port."

http://kubernetes.io/docs/user-guide/services/
Proxy-mode: iptables
- In this mode, kube-proxy watches the Kubernetes master for the addition and
  removal of Service and Endpoints objects.

- For each Service it installs iptables rules which capture traffic to the
  Service’s clusterIP (which is virtual) and Port and redirects that traffic
  to one of the Service’s backend sets.

- For each Endpoints object it installs iptables rules which select a
  backend Pod.

- By default, the choice of backend is random.  Client-IP based session
  affinity can be selected by setting service.spec.sessionAffinity to
  "ClientIP" (the default is "None").

- As with the userspace proxy, the net result is that any traffic bound for
  the Service’s IP:Port is proxied to an appropriate backend without the
  clients knowing anything about Kubernetes or Services or Pods.

  - Service types:

    - ClusterIP
      - Expose a service for connection from inside the cluster.

    - NodePort
      - Expose a service for connection from outside the cluster.

    - LoadBalancer
      -

  - http://kubernetes.io/docs/user-guide/services/#type-nodeport
    "If you set the type field to "NodePort", the Kubernetes master will
     allocate a port from a flag-configured range (default: 30000-32767), and
     each Node will proxy that port (the same port number on every Node) into
     your Service.
     That port will be reported in your Service’s spec.ports[*].nodePort field.

     If you want a specific port number, you can specify a value in the nodePort
     field, and the system will allocate you that port or else the API
     transaction will fail (i.e. you need to take care about possible port
     collisions yourself).  The value you specify must be in the configured
     range for node ports."


- Service Discovery:

- Kubernetes supports 2 primary modes of finding a Service
  - environment variables
    This does imply an ordering requirement - any Service that a Pod wants to
    access must be created before the Pod itself, or else the environment
    variables will not be populated.  DNS does not have this restriction.
  - DNS.
    An optional (though strongly recommended) cluster add-on is a DNS server.
    https://github.com/kubernetes/kubernetes/tree/release-1.2/cluster/addons/dns
    http://www.projectatomic.io/blog/2015/10/setting-up-skydns/

centos-3158596614-s6igh# env | sort | grep ^NGINX
NGINX_SERVICE_PORT=tcp://10.254.252.65:8000
NGINX_SERVICE_PORT_8000_TCP=tcp://10.254.252.65:8000
NGINX_SERVICE_PORT_8000_TCP_ADDR=10.254.252.65
NGINX_SERVICE_PORT_8000_TCP_PORT=8000
NGINX_SERVICE_PORT_8000_TCP_PROTO=tcp
NGINX_SERVICE_SERVICE_HOST=10.254.252.65
NGINX_SERVICE_SERVICE_PORT=8000
-->




## Overview

Explore deploying and running microservices in Kubernetes - a work in progress
though everything pushed here works just fine.




## Scenario

Test environment is as follows:
```
rothko$ cat /etc/redhat-release
CentOS Linux release 7.2.1511 (Core)

rothko$ grep vmx /proc/cpuinfo | wc -l
4

rothko$ free -m
              total        used        free      shared  buff/cache   available
Mem:           7732        1399        4902         455        1430        5564
Swap:          8187           0        8187

rothko$ rpm -q vagrant
vagrant-1.8.1-1.x86_64
```




## Option 1:  Hyperkube Kubernetes deployment

See:
- http://kubernetes.io/docs/getting-started-guides/docker/
- http://kubernetes.io/docs/getting-started-guides/docker-multinode/deployDNS/

Start virtual machine:
```
rothko$ sudo /sbin/iptables -I INPUT 1 -i virbr+ -j ACCEPT

rothko$ cd ~/junio
rothko$ sudo -v ; vagrant up vm1

rothko$ vagrant ssh vm1


vm1$ sudo /bin/systemctl stop firewalld.service
vm1$ sudo /bin/systemctl disable firewalld.service
```

Install Docker Inc's docker-engine 1.11.1 rpm as I couldn't make Hyperkube work
with CentOS' docker 1.9.1 rpm:
```
vm1$ sudo tee /etc/yum.repos.d/docker.repo <<'eof'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/$releasever/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
eof
vm1$ sudo yum -y install docker-engine
vm1$ sudo /bin/systemctl enable docker.service
vm1$ sudo /bin/systemctl start docker.service
```


Configure Docker:
```
vm1$ cd /vagrant/ssl
vm1$ make

vm1$ sudo mkdir -p /etc/docker/certs.d/vm1:5000
vm1$ sudo cp -i \
        /vagrant/ssl/vm1.crt \
        /etc/docker/certs.d/vm1:5000/ca.crt

vm1$ sudo /bin/systemctl restart docker.service
```


Start registry:
```
vm1$ sudo docker run -d -i -t \
	--name registry.docker --hostname registry.docker \
	-p 5000:5000 \
	-v /vagrant/ssl:/certs \
	-v /home/vagrant/registry:/var/lib/registry \
	-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/vm1.crt \
	-e REGISTRY_HTTP_TLS_KEY=/certs/vm1.key \
	registry:2
```


Start Kubernetes:
```
vm1$ curl -sS \
	https://storage.googleapis.com/kubernetes-release/release/stable.txt
v1.2.4
vm1$ curl -sS \
	https://storage.googleapis.com/kubernetes-release/release/latest.txt
v1.3.0-alpha.4

vm1$ sudo docker run -d \
	--volume=/:/rootfs:ro \
	--volume=/sys:/sys:rw \
	--volume=/var/lib/docker/:/var/lib/docker:rw \
	--volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
	--volume=/var/run:/var/run:rw \
	--net=host \
	--pid=host \
	--privileged \
	gcr.io/google_containers/hyperkube-amd64:v1.2.4 \
    		/hyperkube kubelet \
        		--containerized \
        		--hostname-override=127.0.0.1 \
        		--api-servers=http://localhost:8080 \
        		--config=/etc/kubernetes/manifests \
        		--cluster-dns=10.0.0.10 \
			--cluster-domain=cluster.local \
			--allow-privileged --v=2

vm1$ cd /vagrant/misc
vm1$ curl -O http://storage.googleapis.com/kubernetes-release/release/v1.2.0/bin/linux/amd64/kubectl
vm1$ chmod 775 ./kubectl
vm1$ PATH=/vagrant/misc:$PATH

vm1$ kubectl get nodes
NAME        STATUS    AGE
127.0.0.1   Ready     2m

vm1$ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
k8s-etcd-127.0.0.1     1/1       Running   0          1m
k8s-master-127.0.0.1   4/4       Running   4          3m
k8s-proxy-127.0.0.1    1/1       Running   0          2m

vm1$ kubectl get services
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.0.0.1     <none>        443/TCP   3m
```


Run something:
```
vm1$ kubectl run nginx --image=nginx --port=80

	- Wait...

vm1$ kubectl get pods

vm1$ kubectl expose deployment nginx --port=80 --type NodePort

vm1$ kubectl get svc nginx --template='{{(index .spec.ports 0).nodePort}}' \
	; echo
32391

vm1$ curl http://vm1:32391
```


Now DNS:
```
vm1$ cd /vagrant/misc
vm1$ curl -O http://kubernetes.io/docs/getting-started-guides/docker-multinode/skydns.yaml.in

vm1$ DNS_REPLICAS=1
vm1$ DNS_DOMAIN=cluster.local
vm1$ DNS_SERVER_IP=10.0.0.10

vm1$ sed -e "s/{{ pillar\['dns_replicas'\] }}/${DNS_REPLICAS}/g;
	s/{{ pillar\['dns_domain'\] }}/${DNS_DOMAIN}/g;
	s/{{ pillar\['dns_server'\] }}/${DNS_SERVER_IP}/g" \
	< skydns.yaml.in > ./skydns.yaml

vm1$ diff ./skydns.yaml.in ./skydns.yaml
.
.

vm1$ kubectl get ns
NAME      STATUS    AGE
default   Active    18m

vm1$ kubectl create namespace kube-system
namespace "kube-system" created

vm1$ kubectl create -f ./skydns.yaml
replicationcontroller "kube-dns-v10" created
service "kube-dns" created

vm1$ kubectl --namespace kube-system get pods
vm1$ kubectl --namespace kube-system logs kube-dns-v10-29de8
vm1$ kubectl --namespace kube-system logs kube-dns-v10-29de8 etcd
vm1$ kubectl --namespace kube-system logs kube-dns-v10-29de8 kube2sky
vm1$ kubectl --namespace kube-system logs kube-dns-v10-29de8 skydns
vm1$ kubectl --namespace kube-system logs kube-dns-v10-29de8 healthz


vm1$ kubectl run -i --tty chris --image=centos:7 --restart=Never -- /bin/bash

chris-auyn2# yum -y install bind-utils

chris-auyn2# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.0.0.10
options ndots:5

chris-auyn2# dig +short nginx.default.svc.cluster.local in a
10.0.0.105
chris-auyn2# dig +short nginx.default.svc.cluster.local in srv
10 100 0 a0d131fc.nginx.default.svc.cluster.local.
chris-auyn2# getent hosts nginx
10.0.0.105      nginx.default.svc.cluster.local

chris-auyn2# curl http://nginx/
.
.	- Cool.
```




## Option 2:  Cluster Kubernetes deployment

Start virtual machines:
```
rothko$ sudo /sbin/iptables -I INPUT 1 -i virbr+ -j ACCEPT

rothko$ cd ~/junio
rothko$ sudo -v ; vagrant up

rothko$ vagrant ssh vm1
rothko$ vagrant ssh vm2
rothko$ vagrant ssh vm3
rothko$ vagrant ssh vm4


vm1234$ sudo /bin/systemctl stop firewalld
vm1234$ sudo /bin/systemctl disable firewalld
```


Configure etcd - required by flannel:
```
vm1$ sudo yum -y install etcd

vm1$ sudo tee /etc/etcd/etcd.conf <<'eof'
ETCD_NAME=default
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.40.11:2379"
eof

vm1$ sudo /bin/systemctl enable etcd.service
vm1$ sudo /bin/systemctl start etcd.service
```


Configure flannel:
- Note flannel has to be started before Docker, see
  https://github.com/coreos/flannel#docker-integration
- flannel writes /run/flannel/subnet.env which is picked up by
  /usr/lib/systemd/system/docker.service.d/flannel.conf.
```
vm1$ etcdctl mk /junio/network/config '{ "Network" : "172.17.0.0/16" }'
vm1$ etcdctl ls --recursive /junio/network/

vm234$ sudo yum -y install flannel

vm234$ sudo tee /etc/sysconfig/flanneld <<'eof'
FLANNEL_ETCD="http://192.168.40.11:2379"
FLANNEL_ETCD_KEY="/junio/network"
FLANNEL_OPTIONS=""
eof

vm234$ sudo /bin/systemctl enable flanneld.service
vm234$ sudo /bin/systemctl start flanneld.service

vm234$ /sbin/ip addr list flannel0

vm1$ etcdctl ls --recursive /junio/network/
/junio/network/config
/junio/network/subnets
/junio/network/subnets/172.17.25.0-24
/junio/network/subnets/172.17.77.0-24
/junio/network/subnets/172.17.91.0-24
```


Install CentOS' docker 1.9.1 rpm - this is a dependency of CentOS'
kubernetes rpms:
```
vm1234$ sudo yum -y install docker
vm1234$ sudo /bin/systemctl enable docker.service
vm1234$ sudo /bin/systemctl start docker.service
```


Check Docker / flanneld integration - each node should have a different bip:
```
vm234$ ps auxww | grep [d]ocker.*bip
```


Configure Docker:
```
vm1$ cd /vagrant/ssl
vm1$ make

vm1234$ sudo mkdir -p /etc/docker/certs.d/vm1:5000
vm1234$ sudo cp \
        /vagrant/ssl/vm1.crt \
        /etc/docker/certs.d/vm1:5000/ca.crt
vm1234$ sudo /bin/systemctl restart docker.service
```


Start registry:
```
vm1$ sudo docker run -d -i -t \
	--name registry.docker --hostname registry.docker \
	-p 5000:5000 \
	-v /vagrant/ssl:/certs \
	-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/vm1.crt \
	-e REGISTRY_HTTP_TLS_KEY=/certs/vm1.key \
	registry:2

		- Have removed
		  -v /home/vagrant/registry:/var/lib/registry \
		  as was breaking docker push, not sure why.
```


Configure Kubernetes master:
```
vm1$ sudo yum -y install kubernetes

vm1$ sudo tee /etc/kubernetes/apiserver <<'eof'
KUBE_API_ADDRESS="--address=0.0.0.0"
KUBE_API_PORT="--port=8080"
KUBELET_PORT="--kubelet_port=10250"
KUBE_ETCD_SERVERS="--etcd_servers=http://127.0.0.1:2379"
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.0.0.0/16"
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ResourceQuota"
KUBE_API_ARGS=""
eof

vm1$ sudo /bin/systemctl enable \
	kube-apiserver \
	kube-controller-manager \
	kube-scheduler

vm1$ sudo /bin/systemctl start \
	kube-apiserver \
	kube-controller-manager \
	kube-scheduler

vm1$ /bin/systemctl --failed
vm1$ sudo /bin/journalctl
```


Configure Kubernetes nodes / minions:
```
vm234$ sudo yum -y install kubernetes

vm234$ sudo tee /etc/kubernetes/config <<'eof'
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://192.168.40.11:8080"
eof

vm234$ sudo tee /etc/kubernetes/kubelet <<'eof'
KUBELET_ADDRESS="--address=0.0.0.0"
KUBELET_PORT="--port=10250"
#KUBELET_HOSTNAME="--hostname-override=127.0.0.1"
KUBELET_API_SERVER="--api_servers=http://192.168.40.11:8080"
KUBELET_ARGS=""
KUBELET_ARGS="--cluster-dns=10.0.0.10 --cluster-domain=cluster.local"
eof

vm234$ sudo /bin/systemctl enable \
	kubelet \
	kube-proxy

vm234$ sudo /bin/systemctl start \
	kubelet \
	kube-proxy

vm234$ /bin/systemctl --failed
vm234$ sudo /bin/journalctl


vm1$ kubectl get nodes -o wide
NAME      STATUS    AGE
vm2       Ready     1m
vm3       Ready     1m
vm4       Ready     1m

vm1$ kubectl get pods -o wide

vm1$ kubectl get services -o wide
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   10.0.0.1     <none>        443/TCP   4m        <none>
```


Run something:
```
vm1$ kubectl run nginx --image=nginx --port=80

	- Wait...

vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
nginx-198147104-rh8pr   1/1       Running   0          3m        vm4

vm1$ kubectl expose deployment nginx --port=80 --type NodePort

vm1$ kubectl get svc nginx --template='{{(index .spec.ports 0).nodePort}}' \
	; echo
32708

rothko$ curl http://vm2:32708/
rothko$ curl http://vm3:32708/
rothko$ curl http://vm4:32708/
```


Now DNS:
- Here I'm not running the Kubernetes documented configuration at
  http://kubernetes.io/docs/getting-started-guides/docker-multinode/skydns.yaml.in
  but am instead leveraging the existing etcd infrastructure.
```
vm1$ kubectl create -f /vagrant/yaml/skydns-pod.yaml
vm1$ kubectl create -f /vagrant/yaml/skydns-svc.yaml

	- Wait...

vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
nginx-198147104-rh8pr   1/1       Running   0          8m        vm4
skydns                  2/2       Running   0          4m        vm2

vm1$ kubectl get svc -o wide
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE       SELECTOR
kubernetes   10.0.0.1       <none>        443/TCP         13m       <none>
nginx        10.0.123.163   nodes         80/TCP          5m        run=nginx
skydns       10.0.0.10      <none>        53/UDP,53/TCP   3m        name=skydns

vm1$ kubectl logs skydns kube2sky
vm1$ kubectl logs skydns skydns


vm1$ kubectl run -i --tty chris --image=centos:7 --restart=Never -- /bin/bash

chris-72ntz# yum -y install bind-utils

chris-72ntz# cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.0.0.10
nameserver 192.168.121.1
options ndots:5

chris-72ntz# dig +short nginx.default.svc.cluster.local in a
10.0.178.132
chris-72ntz# dig +short nginx.default.svc.cluster.local in srv
10 100 0 7dc693a6.nginx.default.svc.cluster.local.
chris-72ntz# getent hosts nginx
10.0.178.132    nginx.default.svc.cluster.local

chris-72ntz# curl http://nginx/
.
.	- Cool.
```




## Building and publishing microservices

```
vm1$ sudo yum -y install golang

vm1$ cd /vagrant/images/date/
vm1$ rm -f build bin/date ; make build VERSION=1.0.0
vm1$ sudo docker tag junio/date:1.0.0 vm1:5000/junio/date:1.0.0
vm1$ sudo docker push vm1:5000/junio/date:1.0.0

vm1$ cd /vagrant/images/time
vm1$ rm -f build bin/time ; make build VERSION=1.0.0
vm1$ sudo docker tag junio/time:1.0.0 vm1:5000/junio/time:1.0.0
vm1$ sudo docker push vm1:5000/junio/time:1.0.0

vm1$ cd /vagrant/images/web
vm1$ rm -f build bin/web ; make build VERSION=1.0.0
vm1$ sudo docker tag junio/web:1.0.0 vm1:5000/junio/web:1.0.0
vm1$ sudo docker push vm1:5000/junio/web:1.0.0

vm1$ sudo docker images
```




## Deploying microservices

```
vm1$ kubectl create -f /vagrant/yaml/date-pod.yaml
pod "date" created

vm1$ kubectl create -f /vagrant/yaml/date-svc.yaml
service "date" created

vm1$ kubectl create -f /vagrant/yaml/time-pod.yaml
pod "time" created

vm1$ kubectl create -f /vagrant/yaml/time-svc.yaml
service "time" created

vm1$ kubectl create -f /vagrant/yaml/web-pod.yaml
pod "web" created

vm1$ kubectl create -f /vagrant/yaml/web-svc.yaml
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30956) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "web" created


vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
chris-a7esi             1/1       Running   0          10m       vm3
date                    1/1       Running   0          59s       vm3
nginx-198147104-wkp4p   1/1       Running   0          13m       vm4
skydns                  2/2       Running   0          11m       vm2
time                    1/1       Running   0          48s       vm4
web                     1/1       Running   0          42s       vm2
vm1$ kubectl get svc -o wide
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE       SELECTOR
date         10.0.197.3     <none>        7001/TCP        53s       name=date
kubernetes   10.0.0.1       <none>        443/TCP         18m       <none>
nginx        10.0.134.163   nodes         80/TCP          11m       run=nginx
skydns       10.0.0.10      <none>        53/UDP,53/TCP   11m       name=skydns
time         10.0.164.82    <none>        7002/TCP        47s       name=time
web          10.0.156.214   nodes         7000/TCP        40s       name=web
```




## Testing microservices

Testing Cluster Kubernetes deployment:
```
rothko$ curl http://vm2:30956
web :
20160601 - date 
23:16:47 - time 
rothko$ curl http://vm3:30956
web :
20160601 - date 
23:16:52 - time 
rothko$ curl http://vm4:30956
web :
20160601 - date 
23:16:54 - time 
```


Testing Hyperkube Kubernetes deployment:
```
rothko$ curl http://vm1:30956/
web :
20160601 - date 
18:20:36 - time 
rothko$ curl http://vm1:30956/
web :
20160601 - date 
18:20:37 - time 
```


Check DNS:
```
vm1$ kubectl attach chris-a7esi -i -t

chris-syob1# getent hosts date time web
10.0.0.243      date.default.svc.cluster.local
10.0.0.215      time.default.svc.cluster.local
10.0.0.219      web.default.svc.cluster.local

chris-syob1# dig +short _date._tcp.date.default.svc.cluster.local in srv
10 100 7001 date.default.svc.cluster.local.
chris-syob1# dig +short _time._tcp.time.default.svc.cluster.local in srv
10 100 7002 time.default.svc.cluster.local.
chris-syob1# dig +short _web._tcp.web.default.svc.cluster.local in srv
10 100 7000 web.default.svc.cluster.local.

chris-syob1# curl http://time:7002/time ; echo
{"time":"23:12:38","hostname":"time","version":""}
^p^q

	- Awesome.
```




## Querying etcd

This is being used to store configuration for Flannel, Kubernetes and SkyDNS:
```
vm1$ etcdctl ls --recursive / | less
```


Flannel:
```
vm1$ etcdctl get /junio/network/subnets/172.17.77.0-24 | python -m json.tool
```


Kubernetes:
```
vm1$ etcdctl get /registry/pods/default/date | python -m json.tool
vm1$ etcdctl get /registry/services/specs/default/date | python -m json.tool
```


SkyDNS:
```
vm1$ etcdctl get /skydns/local/cluster/default/date | python -m json.tool
vm1$ etcdctl get /skydns/local/cluster/svc/default/date/34cd9f8a | python -m json.tool
```




## Querying

```
vm1$ kubectl get namespaces

vm1$ kubectl get jobs
vm1$ kubectl get deployments
vm1$ kubectl get rc
vm1$ kubectl get pods -o wide
vm1$ kubectl get svc -o wide
```


More detail:
```
vm1$ kubectl get pod nginx-198147104-wkp4p -o json
vm1$ kubectl get svc nginx --template='{{(index .spec.ports 0).nodePort}}'
```


Logs:
```
vm1$ kubectl logs date
```




## Adhoc container with detaching and attaching

```
vm1$ kubectl run -i --tty chris --image=centos:7 --restart=Never -- /bin/bash

chris-wpz4f# ^p^q

vm1$ kubectl get pods -o wide
vm1$ kubectl attach chris-j85bq -i -t

chris-wpz4f# ^d

vm1$ kubectl delete job chris
```




## Exec shell in running container

```
vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
date                    1/1       Running   0          17m       vm3
nginx-198147104-wkp4p   1/1       Running   0          30m       vm4
skydns                  2/2       Running   0          28m       vm2
time                    1/1       Running   0          17m       vm4
web                     1/1       Running   0          17m       vm2

vm1$ kubectl exec -i -t date -- /bin/bash
date# ^d
```




## To do

- Fix flannel path from junio to flannel.
- Need new go to get VERSION strings.
- Fix warning 'Failed to get pwuid struct: user: unknown userid 4294967295'.
- Move skydns to be a replicationcontroller.
- Fix registry /var/lib/registry volume.
- docker-storage-setup.service loaded failed failed Docker Storage Setup
- http://kubernetes.io/docs/hellonode/
- http://kubernetes.io/docs/user-guide/walkthrough/
- http://kubernetes.io/docs/user-guide/walkthrough/k8s201/
- http://rafabene.com/2015/11/11/how-expose-kubernetes-services/
- http://kubernetes.io/docs/user-guide/services/
- http://blog.scottlowe.org/2015/04/15/running-etcd-20-cluster/
