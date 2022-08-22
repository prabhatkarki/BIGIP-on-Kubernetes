# BIGIP-on-Kubernetes


Steps to install Kubernetes/Calico/BGP-setup/Ingress-Controller/AS3 

OS: CentOS
One master one worker Kube nodes
1.	Following commands are run on both master and worker node.
$hostnamectl set-hostname 'k8s-master'
$hostnamectl set-hostname 'worker-node1'
$yum -y update
$reboot
$cat /etc/centos-release
CentOS Linux release 7.7.1908 (Core)

2.	Edit the /etc/hosts file with the IPs that will be used for node IP (incase nodeIPs are different than management IP)
192.168.2.44 k8s-master
192.168.2.48 worker-node1

3.	Disable Firewall
$systemctl disable firewalld
$systemctl stop firewalld

4.	Disable SELinux.
$setenforce 0
$sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux

5.	Enable the br_netfilter module for cluster communication
$modprobe br_netfilter
$echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables

6.	Disable swap to prevent memory allocation issues.
$swapoff -a
 $vi /etc/fstab  ->  Comment out the swap line

7.	Install the Docker prerequisites
$yum install -y yum-utils device-mapper-persistent-data lvm2

8.	Add the Docker repo and install Docker
$yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
$yum install -y docker-ce

9.	Conigure the Docker Cgroup Driver to systemd, enable and start Docker
$sed -i '/^ExecStart/ s/$/ --exec-opt native.cgroupdriver=systemd/' /usr/lib/systemd/system/docker.service 
 $systemctl daemon-reload
 $systemctl enable docker --now
 $systemctl status docker (to ensure that Docker is up)

10.	Add the Kubernetes repo.
$cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
   https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

11.	Install Kubernetes.
$yum install -y kubelet kubeadm kubectl

12.	Enable Kubernetes. The kubelet service will not start until you run kubeadm init.
$systemctl enable kubelet

13.	$reboot

*Note: Complete the following section on the MASTER ONLY!
14.	Initialize the cluster using the IP range for Flannel.
$kubeadm init --pod-network-cidr=172.168.0.0/16

(note save the kubeadm join output to run on worker nodes later)
Run the following as instructed in the output.

 $mkdir -p $HOME/.kube
 $cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 $chown $(id -u):$(id -g) $HOME/.kube/config

15.	Run the following on worker node
$kubeadm join 192.168.1.19:6443 --token r4n4pj.et9165le2fyyp8q5 \
    --discovery-token-ca-cert-hash sha256:39931a83a857d584d505759b603c04864e609917b26835d2719250bb9933b33a

16.	Check the nodes and pods in master
$kubectl get nodes
NAME           STATUS     ROLES    AGE     VERSION
k8s-master     NotReady   master   6m1s    v1.16.2
worker-node1   NotReady   <none>   4m44s   v1.16.2

coredns maybe pending until network overlay CNI Calico is installed
$kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-5644d7b6d9-jf6sc             0/1     Pending   0          7m
kube-system   coredns-5644d7b6d9-xxrpq             0/1     Pending   0          7m
kube-system   etcd-k8s-master                      1/1     Running   0          6m15s
kube-system   kube-apiserver-k8s-master            1/1     Running   0          5m58s
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          6m12s
kube-system   kube-proxy-qcz6m                     1/1     Running   0          6m4s
kube-system   kube-proxy-v2j88                     1/1     Running   0          7m1s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          6m17s

17.	Install Calico On the master node only (all the steps from this point below are on master node only)

$curl https://docs.projectcalico.org/v3.9/manifests/calico.yaml -O
If you are using pod CIDR 192.168.0.0/16, skip to the next step. If you are using a different pod CIDR, use the following commands to set an environment variable called POD_CIDR containing your pod CIDR and replace 192.168.0.0/16 in the manifest with your pod CIDR.
My CIDR is 172.168.0.0/16

$POD_CIDR="172.168.0.0/16" \
> sed -i -e "s?192.168.0.0/16?$POD_CIDR?g" calico.yaml

18.	Apply the manifest using the following command.
$kubectl apply -f calico.yaml

19.	Ensure that all the pods including Calico is running. Proxy and Calico nodes will be running on worker node as well
 

20.	Ensure now that both nodes are ready
$kubectl get nodes
NAME           STATUS   ROLES    AGE    VERSION
k8s-master     Ready    master   152m   v1.16.2
worker-node1   Ready    <none>   151m   v1.16.2

21.	Install calicoctl binary
$curl -O -L  https://github.com/projectcalico/calicoctl/releases/download/v3.10.0/calicoctl

$chmod +x calicoctl
$cp calicoctl /usr/local/bin

22.	Create calicoctl config file
$cat /etc/calico/calicoctl.cfg 
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/root/.kube/config"

23.	Run the calicoctl command to verify
$calicoctl version
Client Version:    v3.10.0
Git commit:        7968b525
Cluster Version:   v3.9.2
Cluster Type:      k8s,bgp,kdd

$calicoctl get nodes
NAME           
k8s-master     
worker-node1   

24.	Setup  BGP
For this you need to have your BIGIP’s selfIP that will be used to talk to kube pods when load balancing traffic across.
On The BigIP ensure that
•	touch /var/config/rest/iapps/enable
•	v16 or later license is installed
•	f5-appsvcs app rpm is uploaded.
•	A partition called Kubernetes is created
•	BGP is assigned to route domain 0 

25.	BGP Setup in BIGIP

$imish
bigip-cis.f5.edu[0]>en
bigip-cis.f5.edu[0]#config terminal
Enter configuration commands, one per line.  End with CNTL/Z.
bigip-cis.f5.edu[0](config)#router bgp 64512
bigip-cis.f5.edu[0](config-router)#neighbor calico-k8s peer-group
bigip-cis.f5.edu[0](config-router)#neighbor calico-k8s remote-as 64512
bigip-cis.f5.edu[0](config-router)#neighbor 192.168.2.44 peer-group calico-k8s
bigip-cis.f5.edu[0](config-router)#neighbor 192.168.2.48 peer-group calico-k8s
bigip-cis.f5.edu[0](config-router)#write 
Building configuration...
[OK]
bigip-cis.f5.edu[0](config-router)#end
bigip-cis.f5.edu[0]#show ip bgp summary 
BGP router identifier 192.168.2.41, local AS number 64512
BGP table version is 1
0 BGP AS-PATH entries
0 BGP community entries

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
192.168.2.44    4 64512       0       1        0    0    0          Active     
192.168.2.48    4 64512       0       1        0    0    0          Active     

Total number of neighbors 2

26.	BGP setup on Calico (master node)
cat << EOF | calicoctl create -f -
> apiVersion: projectcalico.org/v3
> kind: BGPConfiguration
> metadata:
>   creationTimestamp: null
>   name: default
> spec:
>   asNumber: 64512
>   logSeverityScreen: Info
>   nodeToNodeMeshEnabled: true
> EOF
Successfully created 1 'BGPConfiguration' resource(s)

calicoctl get bgpconfig default
NAME      LOGSEVERITY   MESHENABLED   ASNUMBER   
default   Info          true          64512   

27.	Create a global BGP peer with the self IP of BIGIP
cat << EOF | calicoctl create -f -
> apiVersion: projectcalico.org/v3
> kind: BGPPeer
> metadata:
>   name: bigip1
> spec:
>   peerIP: 192.168.2.41
>   asNumber: 64512
> EOF
Successfully created 1 'BGPPeer' resource(s)

28.	Login to BIGIP cli and confirm the kube pods routes have been advertised
igip-cis.f5.edu[0]#show ip r
rip    route  rpf    
bigip-cis.f5.edu[0]#show ip route
Codes: K - kernel, C - connected, S - static, R - RIP, B - BGP
       O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, L1 - IS-IS level-1, L2 - IS-IS level-2, ia - IS-IS inter area
       * - candidate default

C       127.0.0.1/32 is directly connected, lo
C       127.1.1.254/32 is directly connected, tmm
B       192.168.1.0/24 [200/0] via 192.168.2.44, k8s, 00:01:49
C       192.168.2.0/24 is directly connected, k8s
B       192.168.180.192/26 [200/0] via 192.168.2.48, k8s, 00:01:49
B       192.168.235.192/26 [200/0] via 192.168.2.44, k8s, 00:01:49

29.	Pull the CIS image from docker image repo. Need a docker account to pull the CIS image
$docker login
$docker pull 1

30.	Create kube secret bigip-login with BIGIP admin and password
$kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=vzw123

Note:username/password is BIGIP VE’s GUI.

31.	Create the Service Account
$kubectl create serviceaccount k8s-bigip-ctlr -n kube-system

32.	Create Cluster role
$kubectl create clusterrolebinding k8s-bigip-ctlr-clusteradmin --clusterrole=cluster-admin --serviceaccount=kube-system:k8s-bigip-ctlr

33.	Create the CIS pod 
$kubectl create -f setup_cis_bigip1.yaml

The bigip_urL_IP in this yaml not necessarily the mgmt. IP for BIGIP. It’s the tmm selfIP facing the Kubernetes. So change it accordingly
"--bigip-url=10.1.10.20",

34.	Confirm the pod is up
$kubectl get pods --all-namespaces -o wide | grep bigip1
kube-system   k8s-bigip1-ctlr-deployment-78689b9bcb-pjbkw   1/1     Running   0          70s     192.168.180.197   worker-node1   <none>           <none

35.	Deploy the f5-hello-http and f5-hello-https pods. These are two app servers used as pool members for testing. 
$kubectl -f create 1-f5-hello-world-app-http-https-deployment.yaml
$kubectl get pods --all-namespaces -o wide | grep hello


default       f5-hello-world-6df6fcf4fd-l4rjd               1/1     Running   0          4h28m   192.168.180.193   worker-node1   <none>           <none>
default       f5-hello-world-6df6fcf4fd-nzm45               1/1     Running   0          4h28m   192.168.180.195   worker-node1   <none>           <none>
default       f5-hello-world-https-7c95674954-9tm5h         1/1     Running   0          4h28m   192.168.180.194   worker-node1   <none>           <none>
default       f5-hello-world-https-7c95674954-db569         1/1     Running   0          4h28m   192.168.180.196   worker-node1   <none>           <none>


36.	Deploy the AS3 services tied to the f5-hello pods
$kubectl create -f 2-f5-hello-world-app-http-https-service.yaml
$kubectl get service --all-namespaces -o wide | grep hello
default       f5-hello-world         NodePort    10.110.40.91   <none>        8080:30739/TCP           74s     app=f5-hello-world
default       f5-hello-world-https   NodePort    10.97.49.230   <none>        8080:30565/TCP           74s     app=f5-hello-world-https
 

37.	Deploy the AS3 configmap to push AS3 objects to BIGIP
$kubectl create -f 3-f5-as3-configmap-hello-world-http-https.yaml

38.	If everything is working, the BIGIP should have provisioned with AS3 partition and corresponding objects such VS pools etc and it should be marked green.
![image](https://user-images.githubusercontent.com/11083508/186002684-9dfae5a2-acfc-4b50-8dac-552afe748c06.png)
