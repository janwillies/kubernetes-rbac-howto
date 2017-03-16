# How to bring up an RBAC cluster
We will bootstrap a Kubernetes cluster with RBAC permissions and demonstrate different types of permissions (single user and group).

## Bootstrap Kubernetes

We will use `kubeadm` since it's the easiest way to bring up a cluster with rbac. Fortunately for us SIG-Cluster-Lifecycle and SIG-Auth were hard at work and `kubeadm` enables rbac per default and even comes up with a default set of rbac-roles already!

Until Kubernetes-1.6 is out, make sure you have the following
- [Kubernetes-v1.6.0-beta.3](https://dl.k8s.io/v1.6.0-beta.3/kubernetes-server-linux-amd64.tar.gz)
- [kubelet.service](files/kubelet.service) from `files/`

`wget` and `untar` Kubernetes, then copy `kubeadm` to `/usr/bin/`. Copy `files/kubelet.service` to `/etc/systemd/system/` and reload systemd with `systemctl daemon-reload`.

### Master
```
kubeadm init --pod-network-cidr=10.244.0.0/16
```
Note that `10.244.0.0/16` is the default CIDR of flannel, which we will use the overlay network later. If you want to change that, make sure to change flannel as well.
### Worker
scp `files/kubelet.service` to the node and reload systemd with `systemctl daemon-reload`. Then run `kubeadm` with the output from above, e.g.
```
kubeadm join --token fb33d6.9ed5211dbd29a876 10.7.183.62:6443
```

### Checkpoint
Let's check if everything came up correctly, especially the `rbac` related things. On the master:
```
$ export KUBECONFIG=/etc/kubernetes/admin.conf

$ kubectl get node -o wide
NAME       STATUS    AGE       VERSION         EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION
mt02db07   Ready     14h       v1.6.0-beta.2   <none>        Ubuntu 16.04.2 LTS   4.9.13-040913-generic
mt02db08   Ready     14h       v1.6.0-beta.2   <none>        Ubuntu 16.04.2 LTS   4.9.13-040913-generic

$ kubectl get clusterrole
NAME                                           AGE
admin                                          14h
cluster-admin                                  14h
edit                                           14h
system:auth-delegator                          14h
system:basic-user                              14h
system:controller:attachdetach-controller      14h
system:controller:certificate-controller       14h
system:controller:cronjob-controller           14h
[...]
```
For more info about the different roles, have a look at the [RBAC-docs](https://kubernetes-io-vnext-staging.netlify.com/docs/admin/authorization/rbac/). 
Looks good, let's continue with creating the supporting infrastructure
### Network
Create the appropriate `serviceaccount` and `ClusterRole`, then deploy flannel:
```
kubectl create -f files/kube-flannel-rbac.yml
kubectl create clusterrolebinding flannel --clusterrole=flannel --serviceaccount=kube-system:flannel
kubectl create --namespace kube-system -f files/kube-flannel.yml
```

### Tiller
Tiller itself runs with a full access to cluster. But any API call to Tiller via users will use those users authZ level. The latter feature is currently targeted for `helm-v2.3.0` ([relevant PR](https://github.com/kubernetes/helm/pull/1932))
```
kubectl create serviceaccount tiller --namespace=kube-system
kubectl create clusterrolebinding tiller --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init
```
make tiller use the `serviceaccount`:
```
kubectl --namespace=kube-system edit deployment tiller-deploy
```
add `serviceAccount: tiller` to the `spec`-section, e.g.:

```
spec:
  template:
    spec:
      [...]
      restartPolicy: Always
      serviceAccount: tiller
      schedulerName: default-scheduler
      [...]
```

Tiller should restart and have access to all namespaces. Check with `helm ls`, it shouldn't error anymore.

### Deploy an etcd-cluster
This will be used for testing later
```
helm install stable/etcd-operator --namespace=testing --name=etcd-operator
```
Create the necessary `rbac`-rules
```
kubectl create -f files/etcd-operator-rbac.yaml
kubectl --namespace=testing create serviceaccount etcd-operator
kubectl create clusterrolebinding etcd-operator --clusterrole=etcd-operator --serviceaccount=testing:etcd-operator
kubectl --namespace=testing edit deployment etcd-operator
```
Add `serviceAccount: etcd-operator` to the spec:
```
spec:
  template:
    spec:
      [...]
      restartPolicy: Always
      serviceAccount: etcd-operator
      schedulerName: default-scheduler
```
Let the `etcd-operator` create an actual etcd cluster:
```
kubectl --namespace=testing create -f files/etcd-cluster.yaml
```

## Add users
Let's pretend I'm a new user joing the `testing` team. I would need to authenticate myself to the cluster. For the sake of the demo we will use a client cert instead of some of the more advanced methods (e.g. `OIDC`).
### generate client cert
The admin will use openssl to generate a client csr and approve it with the cluster ca.
```
cd /etc/kubernetes/pki
openssl genrsa -out janwillies.key 2048
openssl req -new -key janwillies.key -out janwillies.csr -subj "/CN=janwillies/O=testing/O=preprod"
openssl x509 -req -in janwillies.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out janwillies.crt -days 10000
openssl genrsa -out aricrenzo.key 2048
openssl req -new -key aricrenzo.key -out aricrenzo.csr -subj "/CN=aricrenzo/O=testing"
openssl x509 -req -in aricrenzo.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out aricrenzo.crt -days 10000
```
### kubeconfig
prepare the `kubeconfig` file which we named `janwillies.conf`:
```
base64 -w0 janwillies.crt && echo
base64 -w0 janwillies.key && echo
```
replace `client-certificate-data` and `client-key-data` in `files/janwillies.conf` with the output from above

### test
```
$ export KUBECONFIG=~/janwillies.conf

$ kubectl get pods
Error from server (Forbidden): User "janwillies" cannot list pods in the namespace "etcd". (get pods)
```
oh no!

## Permissions
The `cluster-admin` has to grant access by creating the appropriate `RoleBinding` first:
```
kubectl create -f files/testing-admin-rbac.yaml
```
Notice the `kind: Group` in the file, this means we will grant access to the whole `testing`-team instead of managing roles individually.

Switch back to the user and test again:

```
$ kubectl get pods
NAME                                        READY     STATUS    RESTARTS   AGE
volted-dachshund-etcd-op-1967454838-jr8pb   1/1       Running   0          2h
```

## cleanup
```
sudo su -
systemctl stop kubelet
docker rm -f $(docker ps -a -q)
rm -rf /etc/kubernetes
umount /var/lib/kubelet/pods/*/*/*/*
rm -rf /var/lib/kubelet
rm -rf /var/lib/etcd/
ip addr flush dev cni0
ip addr flush dev flannel.1
exit
```

## Resources
- https://kubernetes-io-vnext-staging.netlify.com/docs/admin/authorization/rbac/
- https://kubernetes.io/docs/admin/authentication/
- https://github.com/kubernetes/helm/pull/1932

## Notes
show `kubelet` logs since last restart of service
```
systemctl show -p ActiveEnterTimestamp kubelet
journalctl -u kubelet --since 2017-03-0823:47:11
```
or directly
```
journalctl -u kubelet --since "$(systemctl show -p ActiveEnterTimestamp kubelet | awk '{print $2 $3}')"
```