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

```​

```sh
kubectl create -f multi-container.yaml​
​
kubectl get po busybox​
```

</p>
</details>





# Question
Create a Pod with main container image "busybox" and which executes the command:​
* “while true; do echo ‘Hi I am from Main container’ >> /var/log/index.html; sleep 5; done” ​
and with sidecar nginx container which exposes the container port 80.​

Use emptyDir Volume and mount it on path:​
* /var/log for busybox​
* /usr/share/nginx/html for nginx ​
Verify both containers are running.​

<details><summary>show</summary>
<p>

Create an initial yaml file with this​

```sh
kubectl run multi-cont-pod --image=busbox --restart=Never --dry-run -o yaml > multi-container.yaml​

​```

Edit the yml as below and create it​

```sh
kubectl create -f multi-container.yamlkubectl get po multi-cont-pod​
```

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  name: multi-container-pod​
spec:​
  containers:​
  - image: busybox​
    name: busybox​
    command:​
    - sh​
    - -c​
    - while true; do echo ‘Hi I am from Main container’ >> /var/log/index.html; sleep 5; done;​
    volumeMounts:​
    - mountPath: /var/log​
      name: empty-dir​
  - image: nginx​
    name: nginx​
    volumeMounts:​
    - mountPath: /usr/share/nginx/html​
      name: empty-dir​
  volumes:​
  - name: empty-dir​
    emptyDir: {}​
```
</p>
</details>

# Question
* Create a multi-container pod using a volume named html, type emptyDir (which means    that the volume is first created when a Pod is assigned to a node and exists as long as that Pod is running on that node). ​
  * The 1st container runs nginx server and has the shared volume mounted to the directory /usr/share/nginx/html. ​
  * The 2nd container uses a debian image and has the shared volume mounted to the directory /html. Every second, the 2nd container adds the current date and time into the index.html file, which is located in the shared volume. ​

Valide your solution​

<details><summary>show</summary>
<p>

A standard use case for a multi-container Pod with a shared Volume is when one container writes logs or other files to the shared directory, and the other container reads from the shared directory. For example, we can create a Pod like so:
​
```sh

cat <<EOF | kubectl apply -f -​
apiVersion: v1​
kind: Pod​
metadata:​
  name: mc1​
spec:​
  volumes:​
  - name: html​
    emptyDir: {}​
  containers:​
  - name: 1st​
    image: nginx​
    volumeMounts:​
    - name: html​
      mountPath: /usr/share/nginx/html​
  - name: 2nd​
    image: debian​
    volumeMounts:​
    - name: html​
      mountPath: /html​
    command: ["/bin/sh", "-c"]​
    args:​
      - while true; do​
          date >> /html/index.html;​
          sleep 1;​
        done​

EOF​
```
​

In this example, we define a volume named html. Its type is emptyDir, which means that the volume is first created when a Pod is assigned to a node, and exists as long as that Pod is running on that node. As the name says, it is initially empty. The 1st container runs nginx server and has the shared volume mounted to the directory /usr/share/nginx/html. The 2nd container uses the Debian image and has the shared volume mounted to the directory /html. Every second, the 2nd container adds the current date and time into the index.html file, which is located in the shared volume. When the user makes an HTTP request to the Pod, the Nginx server reads this file and transfers it back to the user in response to the request.​

​</p>
</details>





# Question
* Create a POD using an Init Container to create a file named “sharedfile.txt” under the “/work” directory and the application container should check if the file exists and sleep for a while. If the file does not exist the application container should exit.

<details><summary>show</summary>
<p>

```yaml
---
apiVersion: v1​
kind: Pod​
metadata:​
name: init-container-test​
spec:​
containers:​
- name: application-container​
image: alpine​
command: ['sh', '-c', 'if [ -f /work/sharedfile.txt ]; then sleep 99999; else exit; fi']​
volumeMounts:​
- name: workdir-volume​
mountPath: /work​
initContainers:​
- name: init-container​
image: busybox:1.28​
command: ['sh', '-c', 'mkdir /work; echo>/work/sharedfile.txt']​
volumeMounts:​
- name: workdir-volume​
mountPath: /work​
volumes:​
- name: workdir-volume​
emptyDir: {}​

```
​</p>
</details>


# Pod Design (20%)​

* Understand how to use Labels, Selectors and Annotations​
  * https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/​
  * https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/​

* Understand Deployments, rolling updates and rollbacks​
  * https://kubernetes.io/docs/concepts/workloads/controllers/deployment/​

* Understand Jobs and CronJobs​
  * https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/​
  * https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/


# Questions

* Create 6 nginx pods in which two of them are labeled env=prod, another two are labeled env=acc, and the last two are labeled env=dev​

<details><summary>show</summary>
<p>
```sh 
kubectl run nginx-dev1 --image=nginx --restart=Never --labels=env=dev​
kubectl run nginx-dev2 --image=nginx --restart=Never --labels=env=dev​
kubectl run nginx-acc1 --image=nginx --restart=Never --labels=env=acc​
kubectl run nginx-acc2 --image=nginx --restart=Never --labels=env=acc​
kubectl run nginx-prod1 --image=nginx --restart=Never --labels=env=prod​
kubectl run nginx-prod2 --image=nginx --restart=Never --labels=env=prod​
```
​​</p>
</details>

* Verify all the pods are created with correct labels​

<details><summary>show</summary>
<p>
```sh 
kubeclt get pods --show-labels​
```
​​</p>
</details>
 ​

* Get the pods with label env=acc​
<details><summary>show</summary>
<p>
```sh 
kubectl get pods -l env=acc​
```
​​</p>
</details>
 ​

* Get the pods with label env​
<details><summary>show</summary>
<p>
```sh 
kubectl get pods -L env​
```
​​</p>
</details>
 ​

* Get the pods with labels env=dev and env=prod and output the labels as well​
<details><summary>show</summary>
<p>
```sh 
kubectl get pods -l 'env in (dev,prod)' --show-labels​
```
​​</p>
</details>
 ​

* Change the label for one of the pod to env=uat and list all the pods to verify​
<details><summary>show</summary>
<p>

```sh 
kubectl label pod/nginx-dev2 env=uat --overwrite​

kubectl get pods --show-labels​
```
​​</p>
</details>
 ​

* Remove the labels for the pods that we created now and verify all the labels are removed​
<details><summary>show</summary>
<p>

```sh 
kubectl label pod nginx-dev{1..2} env-​

kubectl label pod nginx-acc{1..2} env-​

kubectl label pod nginx-prod{1..2} env-​

kubectl get po --show-labels​
```
​​</p>
</details>
 ​

* Let’s add the label app=nginx for all the pods and verify​
<details><summary>show</summary>
<p>

```sh 
kubectl label pod nginx-dev{1..2} app=nginx​

kubectl label pod nginx-acc{1..2} app=nginx​

kubectl label pod nginx-prod{1..2} app=nginx​

kubectl get po --show-labels​
```
​​</p>
</details>
 ​

 ​

* Get all the nodes with labels ​
<details><summary>show</summary>
<p>

```sh 
kubectl get nodes --show-labels​
```
​​</p>
</details>
 ​

* Label the node nodeName=nginxnode​
<details><summary>show</summary>
<p>

```sh 
kubectl label node <nodename> nodeName=nginxnode​
```
​​</p>
</details>
 ​

* Create a Pod that will be deployed on this node with the label nodeName=nginxnode​

<details><summary>show</summary>
<p>

```sh
kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > pod.yaml​
```

Add the nodeSelector like below and create the pod​ ​

```yaml
apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  nodeSelector:​
    nodeName: nginxnode​
  containers:​
  - image: nginx​
    name: nginx​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
``` ​

```sh
kubectl create -f pod.yaml​

```
​​</p>
</details>






​

* Get all the nodes with labels ​
<details><summary>show</summary>
<p>

```sh
kubectl get nodes --show-labels​
```
​​</p>
</details>

Label the node nodeName=nginxnode​

```sh 
kubectl label node <nodename> nodeName=nginxnode​
```
​​</p>
</details>

 ​

* Create a Pod that will be deployed on this node with the label nodeName=nginxnode​
<details><summary>show</summary>
<p>


```sh 
kubectl run nginx --image=nginx --restart=Never --dry-run -o yaml > pod.yaml​
```

Add the nodeSelector like below and create the pod​

 ​
```yaml

apiVersion: v1​
kind: Pod​
metadata:​
  creationTimestamp: null​
  labels:​
    run: nginx​
  name: nginx​
spec:​
  nodeSelector:​
    nodeName: nginxnode​
  containers:​
  - image: nginx​
    name: nginx​
    resources: {}​
  dnsPolicy: ClusterFirst​
  restartPolicy: Never​
status: {}​
```

```sh
kubectl create -f pod.yaml​
```
​​</p>
</details>








 ​

* Create a deployment called webapp with image nginx:1.17.1 with 5 replicas​
<details><summary>show</summary>
<p>


```sh 
kubectl create deployment webapp --image=nginx​
```
​​</p>
</details>
 ​

 ​

* Get the pods of this deployment and their labels​
<details><summary>show</summary>
<p>


```sh 
// get the label of the deployment​

kubectl get deploy --show-labels​

// get the pods with that label​

kubectl get pods -l app=webapp​
```
​​</p>
</details>
 ​

* Scale the deployment from 5 replicas to 10 replicas and verify​
<details><summary>show</summary>
<p>


```sh 
kubectl scale deploy webapp --replicas=10​

kubectl get po -l app=webapp​
```
​​</p>
</details>
 ​

* Get the deployment rollout status​
<details><summary>show</summary>
<p>


```sh 
kubectl rollout status deploy webapp​
```
​​</p>
</details>
 ​

* Get the replicaset that created with this deployment​
<details><summary>show</summary>
<p>


```sh 
kubectl get rs -l app=webapp​
```
​​</p>
</details>
 ​

* Get the yaml of the replicaset and pods of this deployment​
<details><summary>show</summary>
<p>


```sh 
 ​

kubectl get rs -l app=webapp -o yaml​

kubectl get po -l app=webapp -o yaml​
```
​​</p>
</details>
 ​

 ​

* Update the deployment with the image version 1.17.4 and verify​
<details><summary>show</summary>
<p>


```sh 
kubectl set image deploy/webapp nginx=nginx:1.17.4​

kubectl describe deploy webapp | grep Image​

```
​​</p>
</details>

 ​

* Check the rollout history and make sure everything is ok after the update​
<details><summary>show</summary>
<p>


```sh 
kubectl rollout history deploy webapp​

kubectl get deploy webapp --show-labels​

kubectl get rs -l app=webapp​

kubectl get po -l app=webapp​
```
​​</p>
</details>
 ​

 ​

* Undo the deployment to the previous version 1.17.1 and verify Image has the previous version​
<details><summary>show</summary>
<p>


```sh 
kubectl rollout undo deploy webapp​

kubectl describe deploy webapp | grep Image​
```
​​</p>
</details>
 ​

* Update the deployment with the wrong image version 1.100 and verify something is wrong with the deployment​
<details><summary>show</summary>
<p>


```sh 
kubectl set image deploy/webapp nginx=nginx:1.100​

kubectl rollout status deploy webapp (still pending state)​

kubectl get pods (ImagePullErr)​
```
​​</p>
</details>
 ​

* Undo the deployment with the previous version and verify everything is Ok​
<details><summary>show</summary>
<p>


```sh 
kubectl rollout undo deploy webapp​

kubectl rollout status deploy webapp​

kubectl get pods​
```
​​</p>
</details>
 ​

 ​

* Apply the autoscaling to this deployment with minimum 10 and maximum 20 replicas and target CPU of 85% and verify hpa is created and replicas are increased to 10 from 1​
<details><summary>show</summary>
<p>


```sh 
kubectl autoscale deploy webapp --min=10 --max=20 --cpu-percent=85​

kubectl get hpa​

kubectl get pod -l app=webapp​
```
​​</p>
</details>
 ​

 ​

* Clean Up​
<details><summary>show</summary>
<p>


```sh 
kubectl delete deploy webapp​

kubectl delete hpa webapp​
```
​​</p>
</details>
​

* Create a job with an image of busybox which echos “Hello I am from job” and make it run 10 times one after one​
<details><summary>show</summary>
<p>


```sh  ​
kubectl create job hello-job --image=busybox --dry-run -o yaml -- echo "Hello I am from job" > hello-job.yaml​
```
edit the yaml file to add completions: 10​

```yaml
apiVersion: batch/v1​
kind: Job​
metadata:​
  creationTimestamp: null​
  name: hello-job​
spec:​
  completions: 10​
  template:​
    metadata:​
      creationTimestamp: null​
    spec:​
      containers:​
      - command:​
        - echo​
        - Hello I am from job​
        image: busybox​
        name: hello-job​
        resources: {}​
      restartPolicy: Never​
status: {}​
```
 ​
```sh
kubectl create -f hello-job.yaml​
```
 ​

 ​

 ​

* Watch the job that runs 10 times one by one and verify 10 pods are created and delete those after it’s completed​
<details><summary>show</summary>
<p>


```sh 
kubectl get job -w​

kubectl get po​

kubectl delete job hello-job​
```
​​</p>
</details>
 ​

 ​

* Create the same job and make it run 10 times parallel​
<details><summary>show</summary>
<p>


```yaml
apiVersion: batch/v1​
kind: Job​
metadata:​
  creationTimestamp: null​
  name: hello-job​
spec:​
  parallelism: 10​
  template:​
    metadata:​
      creationTimestamp: null​
    spec:​
      containers:​
      - command:​
        - echo​
        - Hello I am from job​
        image: busybox​
        name: hello-job​
        resources: {}​
      restartPolicy: Never​
status: {}​
```
​​</p>
</details>
 ​

 ​

* Create a Cronjob with busybox image that prints date and hello from kubernetes cluster message for every minute​
<details><summary>show</summary>
<p>


```sh 
kubectl create cronjob date-job --image=busybox --schedule="*/1 * * * *" -- bin/sh -c "date; echo Hello from kubernetes cluster"​

```
​​</p>
</details>