# k8s-team-ckad-training

k8s-team-ckad-training

# Requirements

## Docker

[https://docs.docker.com/install/#supported-platforms](https://docs.docker.com/install/#supported-platforms)

## Kubectl

[https://kubernetes.io/docs/tasks/tools/install-kubectl/](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

## kind (Kubernetes in docker)

On Mac via Homebrew:

```sh
brew install kind
```

On Windows:

```powershell
curl.exe -Lo kind-windows-amd64.exe https://github.com/kubernetes-sigs/kind/releases/download/v0.7.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe c:\some-dir-in-your-PATH\kind.exe

OR via [Chocolatey](https://chocolatey.org/packages/kind)
choco install kind -y
```


# Create a kind cluster

```sh
curl https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/cluster-config.yml --silent --output cluster-config.yml

kind create cluster --config cluster-config.yml
```

Deploy calico overlay network (required for the network policy)

```
kubectl apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/calico.yml
kubectl -n kube-system set env daemonset/calico-node FELIX_IGNORELOOSERPF=true
```

Make sure all pods are up and Running before you follow the next steps.

```sh
kubectl get pods --all-namespaces -o wide

NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE     IP             NODE                 NOMINATED NODE   READINESS GATES
kube-system          calico-kube-controllers-5c45f5bd9f-dtx4s     1/1     Running   0          2m31s   192.168.82.1   kind-control-plane   <none>           <none>
kube-system          calico-node-b79hl                            1/1     Running   0          2m30s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          coredns-6955765f44-68xf6                     1/1     Running   0          2m31s   192.168.82.2   kind-control-plane   <none>           <none>
kube-system          coredns-6955765f44-dt8sx                     1/1     Running   0          2m30s   192.168.82.4   kind-control-plane   <none>           <none>
kube-system          etcd-kind-control-plane                      1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-proxy-bknls                             1/1     Running   0          2m30s   172.17.0.2     kind-control-plane   <none>           <none>
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          2m43s   172.17.0.2     kind-control-plane   <none>           <none>
```

# Deploy metrics-server

To enable metrics for CPU and Memory metrics-server has to be installed.
We have prepared a version of metrics-server manifest based on the stable helm chart and updated the flags on the metrics-server container to be able to start in Kind.

```bash
kubectl -n kube-system apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/metrics-server.yml
```

Give it some time and then test if it is working:

```bash
kubectl top nodes
NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kind-control-plane   368m         18%    943Mi           31%

```

Give it some time and then test if it is working:

```bash
kubectl top nodes
NAME                 CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
kind-control-plane   368m         18%    943Mi           31%

```

# Services and Networking (13%)

<https://kubernetes.io/docs/concepts/services-networking/service/>

<https://kubernetes.io/docs/concepts/services-networking/network-policies/>

Create namespace nginx and set is as the current namespace

```sh
kubectl create ns nginx
kubectl config set-context --current --namespace=nginx
```

# Deploy a nginx pod

```sh

kubectl create -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/pod-nginx.yml
```

List all pods with label `app=my-nginx`

```sh
kubectl get pods -l app=my-nginx
```

create service with expose command

```sh
kubectl expose po nginx --port=80 --type=NodePort
kubectl get svc -o wide
```

Find cluster IP

```sh
kubectl get svc -o jsonpath='{.items[0].spec.clusterIP}'
```

# Network policies

create a temporary pod to connect to the nginx nodeport (leave this command running in a new console tab).

```sh
kubectl run curl --image=curlimages/curl --restart=Never -it --rm -- sh -c "while true; do curl --connect-timeout 3 -I <CLUSTER_IP>:80 && sleep 1 ; done"
```

Wait until the previous command returns http status 200 (OK).

Create a network policy to deny all ingress and egress traffic in the current namespace.

```sh
kubectl apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/networkPolicy-default-deny.yml
```

Open the previous tab, where the curl pod command is running, you will probably see a curl error like `curl: (28) Connection timed out after 3001 milliseconds`. This means the network policy is in place and all the inbound/outbound traffic in the namespace is denied.

Create a new network policy to allow egress traffic to port 80 and 443.

```sh

kubectl apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/networkPolicy-allow-egress-http.yml
```

Create another network policy to allow ingress traffic from pod with label `run=curl` to port 80 and 443.

```sh
kubectl apply -f https://raw.githubusercontent.com/schubergphilis/k8s-team-ckad-training/master/networkPolicy-allow-ingress-http-from-curlpod.yml
```

# Dns

<https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/>

Create a nslookup pod

```sh
kubectl run nslookup --image=curlimages/curl --restart=Never -it --rm sh
```

Run the commands bellow inside the nslookup container.

```sh
$ cat /etc/resolv.conf
search nginx.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

$ nslookup 10.96.0.10
Server:  10.96.0.10
Address: 10.96.0.10:53

10.0.96.10.in-addr.arpa name = kube-dns.kube-system.svc.cluster.local

```

As you can see all the dns queries will be forwarded to the kube-dns pod (kube-dns.kube-system.svc.cluster.local).

Resolve the nginx and kubernetes api service address inside the nslookup container.

```
$ nslookup nginx.nginx.svc.cluster.local
Server:  10.96.0.10
Address: 10.96.0.10:53

Name: nginx.nginx.svc.cluster.local
Address: 10.96.204.245

$ nslookup  kubernetes.default.svc.cluster.local
Server:  10.96.0.10
Address: 10.96.0.10:53

Name: kubernetes.default.svc.cluster.local
Address: 10.96.0.1
```

# CKAD Exam preparation by the Schuberg Philis Kubernetes Team

# Who are we?​
* Ivan Dechovski
* Jeferson Vitalino
* Alberto Rodriguez
* Giancarlo Rubio 


# What to expect from the exam?​

* 19 questions in 2 hours.​

* The CKAD is a performance driven exam, speed is key.​

* Allowed documentation domains:​
  * https://kubernetes.io/docs/​
  * https://github.com/kubernetes/​
  * https://kubernetes.io/blog/​

* 66% is a pass.​

* Free re-take ( if booked on cncf)


# Exam tips​

* https://kubernetes.io/docs/reference/kubectl/cheatsheet/​

* Setup Kubectl Autocomplete at the start of the exam!​

* Generate pod/service yaml with kubectl:​

    * kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > pod.yaml​
    * kubectl expose deployment nginx --port=80 --target-port=8000​

* Use kubernetes documentation to find similar YAML manifests.​
* If you are writing YAML, it will slow you down! Copy + paste and Edit.



# Kubernetes Objects​
Kubernetes defines a set of building blocks ("primitives"), which collectively provide mechanisms that deploy, maintain, and scale applications based on CPU, memory or custom metrics.​

* Pods​
* Replica Sets​
* Services​
* Volumes​
* Namespaces​
* ConfigMaps and Secrets​
* StatefulSets​
* DaemonSets​


# Core Concepts (13%)​

* Understand Kubernetes API Primitives​
* Create and Configure Basic Pods

# Core Concepts (13%)​ - Questions

### Create the namespace sbp​
<details><summary>show</summary>
<p>
kubectl create namespace sbp​
</p>
</details>

### ​Using the new namespace, create the nginx pod with version 1.17.8 and expose port 80 of the pod
<details><summary>show</summary>
<p>
kubectl -n sbp run nginx --image=nginx:1.17.8 --restart=Never --port=80
</p>
</details>

### Change the image version to 1.17.8-alpine for the pod you just created and verify the image version is updated​
<details><summary>show</summary>
<p>
kubectl -n sbp set image pod/nginx nginx=nginx:1.17.8-alpine​
kubectl -n sbp describe po nginx​
</p>
</details>

### Change the Image version back to 1.17.8 for the pod you just updated and observe the changes​
<details><summary>show</summary>
<p>
kubectl –n sbp set image pod/nginx nginx=nginx:1.17.1​
kubectl –n sbp describe po nginx​
kubectl –n sbp get po nginx -w # watch it​
</p>
</details>

### Check the Image version without the describe command​
<details><summary>show</summary>
<p>
kubectl get po nginx -o jsonpath='{.spec.containers[].image}{"\n"}'​
​</p>
</details>

### Get the IP Address of the pod you just created​
<details><summary>show</summary>
<p>
kubectl get po nginx -o wide​
kubectl get pod nginx -o jsonpath='{.status.podIP}'​
</p>
</details>

### Create a busy busybox pod that keeps running​
<details><summary>show</summary>
<p>
kubectl run busybox --image=busybox --restart=Never -- /bin/sh -c "sleep 3600"​
</p>
</details>

### Check the connection of the nginx pod from the busybox pod​
<details><summary>show</summary>
<p>
kubectl exec -it busybox -- wget -o- <IP Address>​
kubectl run curl --image=curlimages/curl --restart=Never -it --rm -- sh -c "while true; do curl --connect-timeout 3 -I <POD_IP>:80 && sleep 1 ; done"​
​</p>
</details>

### List the nginx pod with custom columns POD_NAME and POD_STATUS​
<details><summary>show</summary>
<p>
kubectl get pod -o=custom-columns="POD_NAME:.metadata.name, POD_STATUS:.status.containerStatuses[].state"​
</p>
</details>

### List all the pods sorted by name​
<details><summary>show</summary>
<p>
kubectl get pods --sort-by=.metadata.name​
</p>
</details>

### List all the pods sorted by created timestamp​
<details><summary>show</summary>
<p>
kubectl get pods--sort-by=.metadata.creationTimestamp​
</p>
</details>


# Multi-container Pods (10%)
* Understand multi-container pod design patterns (eg: ambassador, adaptor, sidecar)​
* https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/


# Question
Create a Pod with three busy box containers with commands:
* "ls; sleep 3600;"​
* "echo Hello World; sleep 3600;"​
* "echo this is the third container; sleep 3600"​
respectively and check the status.​
* Check the logs of each container that you just created​
* Show metrics of the pod containers and puts them into file.log and verify

<details><summary>show</summary>
<p>

first create single container pod with dry run flag​

```sh
kubectl run busybox --image=busybox --restart=Never --dry-run -o yaml -- bin/sh -c "sleep 3600; ls" > multi-container.yaml​
```

edit the pod to following yaml and create it​

```yaml
​apiVersion: v1​
kind: Pod​
metadata:​
creationTimestamp: null​
labels:​
run: busybox​
name: busybox​
spec:​
containers:​
- args:​
- bin/sh​
- -c​
- ls; sleep 3600​
image: busybox​
name: busybox1​
resources: {}​
- args:​
- bin/sh​
- -c​
- echo Hello world; sleep 3600​
image: busybox​
name: busybox2​
resources: {}​
- args:​
- bin/sh​
- -c​
- echo this is third container; sleep 3600​
image: busybox​
name: busybox3​
resources: {}​
dnsPolicy: ClusterFirst​
restartPolicy: Never​
status: {}​

```

```sh
kubectl create -f multi-container.yaml​
​
kubectl get po busybox​
```

</p>
</details>