# Kubernetes 설치하기

> 실습을 위해서 2019.3월에 사용했던 가상머신 기반에서 Kubernetes를 설치하겠습니다.  
> 가상머신이 없으신 분들은 다음 링크에서 `2. Docker 설치` 까지 먼저 진행하시기 바랍니다.  
> https://github.com/hlkug/meetup/blob/master/201903/%EC%82%AC%EC%A0%84%20%EC%9E%91%EC%97%85.md  

## 1. Kubernetes 설치를 위한 환경 설정

Kubernetes 설치를 위한 환경설정을 node1, node2 모든 VM에 동일하게 적용합니다.

### 1.1 Docker daemon 설정

> 아래 Docker daemon 설정은 root 계정으로 로그인 하여 진행합니다.  
> 모든 VM에 동일하게 적용합니다.

Docker daemon의 cgroup driver를 systemd로 설정하기 위해서 다음의 명령어를 실행합니다.

~~~shell
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
systemctl daemon-reload
systemctl restart docker
~~~


### 1.2 Swap off

다음 명령을 통해서 swap off 시킵니다.
~~~shell
$ sudo swapoff -a
~~~

VM이 재시작되어도 swap을 사용하지 않기 위해서 다음과 같이 `/etc/fatab` 에서 swap 을 주석 처리합니다.
~~~shell
$ sudo vi /etc/fstab
~~~

![Swapoff](https://github.com/hlkug/meetup/raw/master/201904/images/swapoff_fstab.png)


### 1.3 /etc/hosts/ 설정 변경

`/etc/hosts/` 파일을 아래와 같이 모든 VM에 동일하게 적용합니다.

~~~shell
$ sudo vi /etc/hosts
~~~

~~~shell
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

192.168.56.2    node1
192.168.56.3    node2
~~~

### 1.4 routing table 조정

모든 VM에 대해서 다음의 명령을 실행하여 routing table을 조정합니다.

> 다음 명령은 root 계정으로 실행합니다.

~~~shell
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
iptables -A FORWARD -o enp0s3 -j ACCEPT
iptables -A FORWARD -i enp0s3 -j ACCEPT
~~~

## 2. kubeadm, kubelet, kubectl 설치

root로 로그인 하여 다음의 명령을 실행합니다. 
모든 VM에 동일하게 설치합니다.

~~~shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
~~~
 

## 3. Master node 설정

> 마스터 노드는 node1 에서 설정합니다.  
> 일반 계정으로 로그인합니다.

### 3.1 Master node 초기화

`kubeadm init` 명령을 통해서 마스터 노드 초기화를 합니다. k8s 오버레이 네트워크 설정을 위해서 `--pod-network-cidr` 와 현재 실습 VM은 2개의 가상 이더넷 네트워크가 설정되어 있으므로 외부에서 접속가능한 주소를 설정하기 위해서 `--apiserver-advertise-address` 옵션도 함께 입력합니다.

~~~shell
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.56.2 
~~~

위의 명령을 실행하여 정상적으로 master node 초기화가 되었으면 아래와 같은 메시지를 볼 수 있습니다. 

> 아래 결과 메시지의 제일 마지막에 나오는 `kubeadm join` 명령어는 worker node 추가 시 사용할 명령어입니다. 잘 복사해 두시기 바랍니다.  
> 명령어의 token 과 cert-hash 값은 실행하는 환경마다 다른 값으로 생성되니 현 문서의 값이 아닌 작업 중인 환경의 명령어를 복사해야 합니다.

~~~shell
bcadmin@node1:~$ sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.56.2
[init] Using Kubernetes version: v1.14.1
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [node1 localhost] and IPs [192.168.56.2 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [node1 localhost] and IPs [192.168.56.2 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [node1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.56.2]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 17.505337 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node node1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node node1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: fbjacj.sxp9ljgydr1xz0vc
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.56.2:6443 --token fbjacj.sxp9ljgydr1xz0vc \
    --discovery-token-ca-cert-hash sha256:7370e892fbb52ff07062372245bf66bf97954d14a87c971f2046ada96bcd1f4a
~~~

k8s cluster 를 cli인 kubectl로 관리하기 위해서 다음의 명령을 실행하여 config를 설정합니다.

~~~shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

### 3.2 오버레이 네트워크 설정

~~~shell
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
~~~

위의 명령어 실행 후 정상적으로 적용됨을 확인하기 위해서는 아래 명령을 통해서 node의 상태가 `Ready` 인지 확인합니다. 환경에 따라 차이는 있겠지만 몇 초간의 시간이 필요합니다.

~~~shell
kubectl get node
~~~

~~~shell
bcadmin@node1:~$ kubectl get node
NAME    STATUS     ROLES    AGE   VERSION
node1   NotReady   master   17m   v1.14.1
bcadmin@node1:~$ kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml

clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
bcadmin@node1:~$ kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
configmap/calico-config created
service/calico-typha created
deployment.apps/calico-typha created
poddisruptionbudget.policy/calico-typha created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
bcadmin@node1:~$ kubectl get node
NAME    STATUS   ROLES    AGE   VERSION
node1   Ready    master   18m   v1.14.1
bcadmin@node1:~$
~~~

여기까지 진행하였으면 master node 초기화 및 네트워크 설정은 완료되었습니다. 다음 단계로 Worker node 추가만하면 기초적인 k8s 환경 구성이 완료됩니다.

## 4. Worker node 추가

Worker node 추가는 node2에서 진행을 합니다. `1. Kubernetes 설치를 위한 환경 설정` 과 `2. kubeadm, kubelet, kubectl 설치` 이 완료되었는지 다시 한 번 확인합니다.  

앞서 master node 초기화 명령 실행 후 콘솔에 출력되었던 `kubeadm join ` 명령어를 복사하여 node2에서 다음과 같이 실행합니다.

> 앞서 설명한 것과 같이 아래 명령에서 `token` 과 `discovery-token-ca-cert-hash` 값은 실행하는 환경마다 틀리므로 현 문서의 값이 아니라 실행 중인 VM에서의 값을 사용하여야 합니다.

~~~shell
kubeadm join 192.168.56.2:6443 --token fbjacj.sxp9ljgydr1xz0vc \
    --discovery-token-ca-cert-hash sha256:7370e892fbb52ff07062372245bf66bf97954d14a87c971f2046ada96bcd1f4a
~~~

정상적으로 명령이 실행되면 다음과 같은 메시지를 콘솔에서 확인 할 수 있습니다.

~~~shell
bcadmin@node2:~$ sudo kubeadm join 192.168.56.2:6443 --token fbjacj.sxp9ljgydr1xz0vc \
>     --discovery-token-ca-cert-hash sha256:7370e892fbb52ff07062372245bf66bf97954d14a87c971f2046ada96bcd1f4a
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
~~~

Master node (node1) 에서 다음의 명령을 실행하여 정상적으로 worker node가 추가되었는지 확인합니다.

~~~shell
bcadmin@node1:~$ kubectl get node
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   5m31s   v1.14.1
node2   Ready    <none>   2m5s    v1.14.1
~~~
