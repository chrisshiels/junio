# junio


<!--
- Notes:

- Kubernetes Cluster:

  - Docker:
    - "an application container technology."

  - Etcd:
    - "a distributed key-value datastore that manages cluster-wide information
       and provides service discovery.



  - etcd.

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


- Concepts:

  - Services:
    - Collections of pods that are exposed with a single and stable name and
      network address.
    - The service provides load balancing to the underlying pods,
      with or without an external load balancer.


- Networking:

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

Explore blue / green microservice deployment in Kubernetes.




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




## Notes - Concepts

- Pod:
  - Group of one or more containers running and sharing resources on the
    same Kubernetes node.

- Label:
  - Key-value pairs used to organise pods into groups.

- Replication Controller:
  - Defines a Pod creation template and desired replica count.
  - Automatically creates and kills pods as necessary.

- Deployment:
  - Defines a Pod creation template and desired replica count.
  - Supports declarative updates and can be used to handle rolling updates
    of new image versions.

- Service:
  - Provides a single IP to refer to a set of Pods selected by labels.
  - Three different types:
    - ClusterIP
      - Default.
      - Reachable from inside the cluster only.
    - NodePort
      - Configures a ClusterIP and exposes the service on all cluster nodes
        at the same port.
    - LoadBalancer:
      - Configures a NodePort and requests cloud provider create a load
        balancer.




## Notes - flannel

- See:  https://github.com/coreos/flannel

- Overlay network used with Kubernetes to support PodIPs, i.e. each pod has
  it's own IP address.

- Alternatives include Weave Net.

- Configuration and state stored in etcd.

- Configuration includes network range to use for overlay network, e.g.
  172.17.0.0/16.

- Each host runs flanneld process, self-allocates a /24 subnet from the overlay
  network, e.g. 172.17.24.0/16, configures the flannel0 device and stores
  state in etcd, e.g. /flannel/network/subnets/172.17.25.0-24.

- Implemented through different backends: udp, vxlan, aws-vpc, etc.




## Notes - Networking

- PodIP
  - Implemented via flannel using network range configured in etcd.

- ClusterIP
  - Implemented via iptables using network range configured in
    /etc/kubernetes/apiserver setting KUBE_SERVICE_ADDRESSES:
```
-A KUBE-SEP-3SYDRZYPSRWRWL4L -s 172.17.24.5/32 -m comment --comment "default/time-1-0-0:time" -j KUBE-MARK-MASQ
-A KUBE-SEP-3SYDRZYPSRWRWL4L -p tcp -m comment --comment "default/time-1-0-0:time" -m tcp -j DNAT --to-destination 172.17.24.5:7002
-A KUBE-SEP-72RMBGD2NVGMJYUB -s 172.17.75.5/32 -m comment --comment "default/time-1-0-0:time" -j KUBE-MARK-MASQ
-A KUBE-SEP-72RMBGD2NVGMJYUB -p tcp -m comment --comment "default/time-1-0-0:time" -m tcp -j DNAT --to-destination 172.17.75.5:7002
-A KUBE-SEP-NTHXK4HOYZTL553L -s 172.17.55.4/32 -m comment --comment "default/time-1-0-0:time" -j KUBE-MARK-MASQ
-A KUBE-SEP-NTHXK4HOYZTL553L -p tcp -m comment --comment "default/time-1-0-0:time" -m tcp -j DNAT --to-destination 172.17.55.4:7002
-A KUBE-SERVICES -d 10.0.220.159/32 -p tcp -m comment --comment "default/time-1-0-0:time cluster IP" -m tcp --dport 7002 -j KUBE-SVC-RCVCLTNF2OHHYZAH
-A KUBE-SVC-RCVCLTNF2OHHYZAH -m comment --comment "default/time-1-0-0:time" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-3SYDRZYPSRWRWL4L
-A KUBE-SVC-RCVCLTNF2OHHYZAH -m comment --comment "default/time-1-0-0:time" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-NTHXK4HOYZTL553L
-A KUBE-SVC-RCVCLTNF2OHHYZAH -m comment --comment "default/time-1-0-0:time" -j KUBE-SEP-72RMBGD2NVGMJYUB
```

- NodePort
  - Implemented via iptables:
```
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/time-1-0-0:time" -m tcp --dport 30879 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/time-1-0-0:time" -m tcp --dport 30879 -j KUBE-SVC-RCVCLTNF2OHHYZAH
```




## Notes - etcd

- "etcd is a distributed key value store that provides a reliable way to store
   data across a cluster of machines."


- Ports:
  - etcd-client:  tcp/2379  etcd client communication.
  - etcd-server:  tcp/2380  etcd server to server / peer communication.
  - tcp/4001 and tcp/7001 are legacy.


- See:
  - https://coreos.com/etcd/docs/latest/clustering.html
  - https://coreos.com/os/docs/latest/cluster-architectures.html
  - https://www.youtube.com/watch?v=duUTk8xxGbU
    "CoreOS: Bootstrapping etcd"


- Clustering options are:
  - Static.
  - etcd Discovery.
  - DNS Discovery.


- Here I'm using static discovery and suspect DNS Discovery would work well
  in the cloud for non-asg instances.


- https://coreos.com/etcd/docs/latest/configuration.html#clustering-flags
  - "-initial prefix flags are used in bootstrapping (static bootstrap,
     discovery-service bootstrap or runtime reconfiguration) a new member,
     and ignored when restarting an existing member."


- https://coreos.com/etcd/docs/latest/faq.html#how-does---endpoint-work-with-etcdctl
  - How does -endpoint work with etcdctl?
    - "If only one peer is specified via the -endpoint flag, the etcdctl
       discovers the rest of the cluster via the member list of that one peer,
       and then it randomly chooses a member to use."

    - "Note: -peers flag is now deprecated and -endpoint should be used
       instead, as it might confuse users to give etcdctl a peerURL.




## Build templater utility

The Kubernetes kubectl utility doesn't currently have any templating support
so the following utility is here as a workaround.
```
vm1$ sudo yum -y install golang

vm1$ cd /vagrant/templater
vm1$ make
```




## Option 1:  Hyperkube Kubernetes deployment

"hyperkube is an all-in-one binary for the Kubernetes server components."

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


Configure etcd - required by Flannel, Kubernetes and SkyDNS:
```
vm1234$ sudo yum -y install etcd


vm2$ sudo tee /etc/etcd/etcd.conf <<eof
ETCD_NAME="vm2"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"

ETCD_INITIAL_CLUSTER="vm2=http://192.168.40.12:2380,vm3=http://192.168.40.13:2380,vm4=http://192.168.40.14:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcdcluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.40.12:2380"

ETCD_LISTEN_PEER_URLS="http://192.168.40.12:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.40.12:2379"
eof


vm3$ sudo tee /etc/etcd/etcd.conf <<eof
ETCD_NAME="vm3"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"

ETCD_INITIAL_CLUSTER="vm2=http://192.168.40.12:2380,vm3=http://192.168.40.13:2380,vm4=http://192.168.40.14:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcdcluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.40.13:2380"

ETCD_LISTEN_PEER_URLS="http://192.168.40.13:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.40.13:2379"
eof


vm4$ sudo tee /etc/etcd/etcd.conf <<eof
ETCD_NAME="vm4"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"

ETCD_INITIAL_CLUSTER="vm2=http://192.168.40.12:2380,vm3=http://192.168.40.13:2380,vm4=http://192.168.40.14:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcdcluster"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.40.14:2380"

ETCD_LISTEN_PEER_URLS="http://192.168.40.14:2380"
ETCD_LISTEN_CLIENT_URLS="http://0.0.0.0:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.40.14:2379"
eof


vm234$ sudo /bin/systemctl enable etcd.service
vm234$ sudo /bin/systemctl start etcd.service
vm234$ sudo /bin/systemctl status etcd.service
```


Check:
```
vm1234$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 member list
vm1234$ ETCDCTL_ENDPOINT=http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 etcdctl cluster-health
vm234$ curl -s http://localhost:2379/v2/stats/self | python -mjson.tool
vm234$ curl -s http://localhost:2379/v2/members | python -mjson.tool
```




Configure flannel:
- See:
  - https://github.com/coreos/flannel
- Note flannel has to be started before Docker, see
  https://github.com/coreos/flannel#docker-integration
- flannel writes /run/flannel/docker which is picked up by
  /usr/lib/systemd/system/docker.service.d/flannel.conf.
```
vm1$ etcdctl -endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 mk /flannel/network/config '{
  "Network": "172.17.0.0/16",
  "Backend": {
    "Type": "udp",
    "Port": 8285
  }
}'
vm1$ etcdctl -endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 ls --recursive /flannel/network/
vm1$ etcdctl -endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 get /flannel/network/config

vm234$ sudo yum -y install flannel

vm234$ sudo tee /etc/sysconfig/flanneld <<'eof'
FLANNEL_ETCD="http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379"
FLANNEL_ETCD_KEY="/flannel/network"
FLANNEL_OPTIONS=""
eof

vm234$ sudo /bin/systemctl enable flanneld.service
vm234$ sudo /bin/systemctl start flanneld.service
vm234$ sudo /bin/systemctl status flanneld.service

vm234$ /sbin/ip addr list flannel0

vm1$ etcdctl -endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 ls --recursive /flannel/network/
/flannel/network/config
/flannel/network/subnets
/flannel/network/subnets/172.17.25.0-24
/flannel/network/subnets/172.17.77.0-24
/flannel/network/subnets/172.17.91.0-24
```


Install CentOS' docker 1.9.1 rpm - this is a dependency of CentOS'
kubernetes rpms:
```
vm1234$ sudo yum -y install docker
vm1234$ sudo /bin/systemctl enable docker.service
vm1234$ sudo /bin/systemctl start docker.service
vm1234$ sudo /bin/systemctl status docker.service
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
vm1234$ sudo /bin/systemctl status docker.service
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
KUBE_ETCD_SERVERS="--etcd_servers=http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379"
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

vm1$ sudo /bin/systemctl status \
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

vm234$ sudo /bin/systemctl status \
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
- Note I'm not running the Kubernetes documented configuration at
  http://kubernetes.io/docs/getting-started-guides/docker-multinode/skydns.yaml.in
  but am instead leveraging the existing etcd infrastructure.

- Note also that I've not been able to find a version of kube2sky that lets
  you specify multiple etcd servers...
```
vm1$ /vagrant/templater/templater \
	ETCD_INITIAL_CLUSTER="vm2=http://192.168.40.12:2380,vm3=http://192.168.40.13:2380,vm4=http://192.168.40.14:2380" \
	/vagrant/yaml/skydns.yaml.template | kubectl create -f /dev/stdin

	- Wait...

vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
nginx-198147104-nzkkm   1/1       Running   0          7m        vm4
skydns-ogs0l            3/3       Running   0          3m        vm2
skydns-q824g            3/3       Running   0          2m        vm3
skydns-xvub4            3/3       Running   0          2m        vm4

vm1$ kubectl get svc -o wide
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE       SELECTOR
kubernetes   10.0.0.1       <none>        443/TCP         13m       <none>
nginx        10.0.123.163   nodes         80/TCP          5m        run=nginx
skydns       10.0.0.10      <none>        53/UDP,53/TCP   3m        name=skydns

vm1$ kubectl logs skydns-ogs0l etcd
vm1$ kubectl logs skydns-ogs0l kube2sky
vm1$ kubectl logs skydns-ogs0l skydns


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
^p^q
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
vm1$ /vagrant/templater/templater \
	VERSION=1.0.0 \
	/vagrant/yaml/date.yaml.template | kubectl create -f /dev/stdin
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30155) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "date-1-0-0" created
replicationcontroller "date-1-0-0" created


vm1$ /vagrant/templater/templater \
	VERSION=1.0.0 \
	/vagrant/yaml/time.yaml.template | kubectl create -f /dev/stdin
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30848) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "time-1-0-0" created
replicationcontroller "time-1-0-0" created


vm1$ /vagrant/templater/templater \
	VERSION=1.0.0 \
	/vagrant/yaml/web.yaml.template | kubectl create -f /dev/stdin
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:30373) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "web-1-0-0" created
replicationcontroller "web-1-0-0" created
```


Check:
```
vm1$ kubectl get svc
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
date-1-0-0   10.0.210.151   nodes         7001/TCP        1m
kubernetes   10.0.0.1       <none>        443/TCP         46m
nginx        10.0.171.130   nodes         80/TCP          30m
skydns       10.0.0.10      <none>        53/UDP,53/TCP   29m
time-1-0-0   10.0.23.233    nodes         7002/TCP        1m
web-1-0-0    10.0.232.251   nodes         7000/TCP        57s

vm1$ kubectl get rc
NAME         DESIRED   CURRENT   AGE
date-1-0-0   3         3         1m
skydns       3         3         29m
time-1-0-0   3         3         1m
web-1-0-0    3         3         1m

vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
chris-tzgdh             1/1       Running   0          25m       vm3
date-1-0-0-18sym        1/1       Running   0          1m        vm3
date-1-0-0-4mwxt        1/1       Running   0          1m        vm4
date-1-0-0-nnwcj        1/1       Running   0          1m        vm2
nginx-198147104-nzkkm   1/1       Running   0          34m       vm4
skydns-ogs0l            3/3       Running   0          29m       vm2
skydns-q824g            3/3       Running   0          28m       vm3
skydns-xvub4            3/3       Running   0          28m       vm4
time-1-0-0-64tcs        1/1       Running   0          1m        vm4
time-1-0-0-gjq8c        1/1       Running   0          1m        vm2
time-1-0-0-jg877        1/1       Running   0          1m        vm3
web-1-0-0-390ml         1/1       Running   0          1m        vm4
web-1-0-0-51rc9         1/1       Running   0          1m        vm3
web-1-0-0-mt3k4         1/1       Running   0          1m        vm2
```




## Update DNS
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 set /skydns/local/cluster/svc/default/date '{
  "host": "date-1-0-0.default.svc.cluster.local.",
  "port": 7001
}'
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 set /skydns/local/cluster/svc/default/time '{
  "host": "time-1-0-0.default.svc.cluster.local.",
  "port": 7002
}'
```


Check:
```
vm1$ kubectl attach chris-f1o7w -i -t

chris-f1o7w# dig +short date.default.svc.cluster.local in srv
10 100 7001 date-1-0-0.default.svc.cluster.local.
chris-f1o7w# dig +short time.default.svc.cluster.local in srv
10 100 7002 time-1-0-0.default.svc.cluster.local.

chris-f1o7w# curl http://date-1-0-0:7001/date ; echo
{"date":"20160608","hostname":"date-1-0-0-nnwcj","version":"1.0.0"}
^p^q
```




## Testing microservices

Query the port:
```
vm1$ kubectl get svc web-1-0-0 \
	--template='{{(index .spec.ports 0).nodePort}}' ; echo
30373
```


Testing Cluster Kubernetes deployment:
```
rothko$ curl http://vm2:30373
web-1-0-0-390ml 1.0.0:
20160608 - date-1-0-0-4mwxt 1.0.0
23:27:00 - time-1-0-0-jg877 1.0.0
rothko$ curl http://vm3:30373
web-1-0-0-mt3k4 1.0.0:
20160608 - date-1-0-0-18sym 1.0.0
23:27:05 - time-1-0-0-gjq8c 1.0.0
rothko$ curl http://vm4:30373
web-1-0-0-51rc9 1.0.0:
20160608 - date-1-0-0-nnwcj 1.0.0
23:27:07 - time-1-0-0-64tcs 1.0.0
```


Testing Hyperkube Kubernetes deployment:
```
rothko$ curl http://vm1:30956/
web-i2fek 1.0.0:
20160608 - date-llmg7 1.0.0
17:05:29 - time-v3byr 1.0.0
rothko$ curl http://vm1:30956/
web-i2fek 1.0.0:
20160608 - date-llmg7 1.0.0
17:05:29 - time-v3byr 1.0.0
```




## Building and publishing new version of date microservice
```
vm1$ cd /vagrant/images/date/
vm1$ rm -f build bin/date ; make build VERSION=1.0.1
vm1$ sudo docker tag junio/date:1.0.1 vm1:5000/junio/date:1.0.1
vm1$ sudo docker push vm1:5000/junio/date:1.0.1
```




## Deploying new version of date microservice
```
vm1$ /vagrant/templater/templater \
	VERSION=1.0.1 \
	/vagrant/yaml/date.yaml.template | kubectl create -f /dev/stdin
You have exposed your service on an external port on all nodes in your
cluster.  If you want to expose this service to the external internet, you may
need to set up firewall rules for the service port(s) (tcp:32180) to serve traffic.

See http://releases.k8s.io/release-1.2/docs/user-guide/services-firewalls.md for more details.
service "date-1-0-1" created
replicationcontroller "date-1-0-1" created
```


Check:
```
vm1$ kubectl get svc
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
date-1-0-0   10.0.210.151   nodes         7001/TCP        7m
date-1-0-1   10.0.89.241    nodes         7001/TCP        1m
kubernetes   10.0.0.1       <none>        443/TCP         52m
nginx        10.0.171.130   nodes         80/TCP          36m
skydns       10.0.0.10      <none>        53/UDP,53/TCP   35m
time-1-0-0   10.0.23.233    nodes         7002/TCP        7m
web-1-0-0    10.0.232.251   nodes         7000/TCP        7m

vm1$ kubectl get rc
NAME         DESIRED   CURRENT   AGE
date-1-0-0   3         3         8m
date-1-0-1   3         3         1m
skydns       3         3         36m
time-1-0-0   3         3         7m
web-1-0-0    3         3         7m

vm1$ kubectl get pods -o wide
NAME                    READY     STATUS    RESTARTS   AGE       NODE
chris-tzgdh             1/1       Running   0          31m       vm3
date-1-0-0-18sym        1/1       Running   0          8m        vm3
date-1-0-0-4mwxt        1/1       Running   0          8m        vm4
date-1-0-0-nnwcj        1/1       Running   0          8m        vm2
date-1-0-1-vcsjr        1/1       Running   0          2m        vm2
date-1-0-1-zon5j        1/1       Running   0          2m        vm4
date-1-0-1-zra2s        1/1       Running   0          2m        vm3
nginx-198147104-nzkkm   1/1       Running   0          40m       vm4
skydns-ogs0l            3/3       Running   0          36m       vm2
skydns-q824g            3/3       Running   0          35m       vm3
skydns-xvub4            3/3       Running   0          35m       vm4
time-1-0-0-64tcs        1/1       Running   0          8m        vm4
time-1-0-0-gjq8c        1/1       Running   0          8m        vm2
time-1-0-0-jg877        1/1       Running   0          8m        vm3
web-1-0-0-390ml         1/1       Running   0          7m        vm4
web-1-0-0-51rc9         1/1       Running   0          7m        vm3
web-1-0-0-mt3k4         1/1       Running   0          7m        vm2
```




## Update DNS
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 set /skydns/local/cluster/svc/default/date '{
  "host": "date-1-0-1.default.svc.cluster.local.",
  "port": 7001
}'
```


Check:
```
vm1$ kubectl attach chris-f1o7w -i -t

chris-f1o7w# dig +short date.default.svc.cluster.local in srv
10 100 7001 date-1-0-1.default.svc.cluster.local.
chris-f1o7w# dig +short time.default.svc.cluster.local in srv
10 100 7002 time-1-0-0.default.svc.cluster.local.

chris-f1o7w# curl http://date-1-0-0:7001/date ; echo
{"date":"20160608","hostname":"date-1-0-0-4mwxt","version":"1.0.0"}
chris-f1o7w# curl http://date-1-0-1:7001/date ; echo
{"date":"20160608","hostname":"date-1-0-1-vcsjr","version":"1.0.1"}
^p^q

	- Awesome.
```




## Testing microservices

Testing Cluster Kubernetes deployment:
```
rothko$ curl http://vm2:30373
web-1-0-0-390ml 1.0.0:
20160608 - date-1-0-1-zon5j 1.0.1
23:36:47 - time-1-0-0-jg877 1.0.0
rothko$ curl http://vm3:30373
web-1-0-0-390ml 1.0.0:
20160608 - date-1-0-1-zon5j 1.0.1
23:36:49 - time-1-0-0-jg877 1.0.0
rothko$ curl http://vm4:30373
web-1-0-0-51rc9 1.0.0:
20160608 - date-1-0-1-zra2s 1.0.1
23:36:51 - time-1-0-0-64tcs 1.0.0
```




## Stopping date-1-0-0 microservice
```
vm1$ /vagrant/templater/templater \
	VERSION=1.0.0 \
	/vagrant/yaml/date.yaml.template | kubectl delete -f /dev/stdin
```




## Querying etcd

This is being used to store configuration for Flannel, Kubernetes and SkyDNS:
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 ls --recursive / | sort | less
```


Flannel:
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 get /flannel/network/subnets/172.17.77.0-24 | python -m json.tool
```


Kubernetes:
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 get /registry/controllers/default/date-1-0-1 | \
	python -m json.tool
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 get /registry/services/specs/default/date-1-0-1 | \
	python -m json.tool
```


SkyDNS:
```
vm1$ etcdctl --endpoint http://192.168.40.12:2379,http://192.168.40.13:2379,http://192.168.40.14:2379 get /skydns/local/cluster/svc/default/date-1-0-1/4a918566 | \
	python -m json.tool
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
vm1$ kubectl get svc nginx \
	--template='{{(index .spec.ports 0).nodePort}}' ; echo
```


Logs:
```
vm1$ kubectl logs nginx-198147104-ymsjr
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
vm1$ kubectl get pods -o wide -l name=date-1-0-1
NAME               READY     STATUS    RESTARTS   AGE       NODE
date-1-0-1-vcsjr   1/1       Running   0          12m       vm2
date-1-0-1-zon5j   1/1       Running   0          12m       vm4
date-1-0-1-zra2s   1/1       Running   0          12m       vm3

vm1$ kubectl exec -i -t date-1-0-1-vcsjr -- /bin/bash
date-1-0-1-0kerr# ^d
```




## To do

- Fix warning 'Failed to get pwuid struct: user: unknown userid 4294967295'.
- Fix registry /var/lib/registry volume.
- docker-storage-setup.service loaded failed failed Docker Storage Setup
- http://kubernetes.io/docs/user-guide/services/
- See:  http://blog.kubernetes.io/2016/06/illustrated-childrens-guide-to-kubernetes.html




<!--
create -f
delete -f

Do services before pods/rc/deployments - though not sure why:
"Create service before corresponding replication controllers so that the
scheduler can spread the pods comprising the service."
"create a Service before corresponding Deployments so that the
scheduler can spread the pods comprising the service."

Note I'm querying ClusterIPs, not PodIPs...

chris-f1o7w# dig +short date-1-0-0.default.svc.cluster.local in srv
10 100 0 a4de8a58.date-1-0-0.default.svc.cluster.local.
chris-f1o7w# dig +short _date._tcp.date-1-0-0.default.svc.cluster.local in srv
10 100 7001 date-1-0-0.default.svc.cluster.local.
-->
