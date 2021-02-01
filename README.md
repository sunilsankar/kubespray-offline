# kubespray-offline

1. untar the folder

`tar -zxvf kubespray-offline.tgz`

2. Copy to root directory

`cp /root/kubespray-offline/packages.tgz /root/`

3. Change the permission

`tar -zxvf /root/packages.tgz && chmod 755 /root/
&& chmod -R 755 /root/packages`

4. add the line in /etc/apt/sources.list

`deb [trusted=yes] file:/root/packages/amd64/ /`

5. Install python3 and all the necessary packages

`apt update && apt install git python3 python3-pip apache2 -y`

6. Now copy the packages directory under /var/www/html

`cp -Rvf /root/packages /var/www/html/`

7. Now add the following lines /etc/apt/sources.list

`deb [trusted=yes] http://<kubemasterip>/packages/amd64/ /`

`deb [trusted=yes] http://<kubemasterip>/packages/docker-ce/ /`
### Example
`deb [trusted=yes] http://192.168.8.191/packages/amd64/ /`

`deb [trusted=yes] http://192.168.8.191/packages/docker-ce/ /`

8. Install docker 

`apt update && apt install docker-ce* -y`

9. Copy the daemon.json which has the internal registry

`cd /root/kubespray-offline && cp daemon.json /etc/docker/`

10. Modify the ipaddress 
  
`{
  "insecure-registries": ["192.168.8.191:5000"]
}
`

11. Restart docker service

`systemctl restart docker && systemctl enable docker`

> Please follow 7,8,9,10,11 steps in all the kubernetes nodes

12. Now in master node , need to install the necessary pip modules

`tar -zxvf /root/kubespray-offline/pip-packages.tgz && pip3 install --no-index --find-links="/root/kubespray-offline/pip-packages" -r  /root/kubespray-offline/kubespray/requirements.txt`

13. Now to configure the internal registry . All the necessary folder and images are present here . First load the docker registry image

`cd /root/kubespray-offline `

`docker load < registry-2.tar`

14. Unzip the volume zip file

`tar -zxvf register-mount.tgz`

`mv registry /mnt/`

`chmod -R  755 /mnt/registry/`

15. Start the docker registry

> docker run -d \
  -p 5000:5000 \
  --restart=always \
  --name registry \
  -v /mnt/registry:/var/lib/registry \
  registry:2

16. Now to install the kubernetes using kubespray

`cp -rfp inventory/sample inventory/mycluster`

`cp -rfp inventory/sample inventory/mycluster`
### Update Ansible inventory file with inventory builder
>declare -a IPS=(192.168.8.191 192.168.8.206 192.168.8.88 192.168.8.72s)

17. Build the inventory
> CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

18. Modify the inventory file since we only need 1 master
``` yaml
all:
 hosts:
   node1:
     ansible_host: 192.168.8.191
     ip: 192.168.8.191
     access_ip: 192.168.8.191
   node2:
     ansible_host: 192.168.8.206
     ip: 192.168.8.206
     access_ip: 192.168.8.206
   node3:
     ansible_host: 192.168.8.88
     ip: 192.168.8.88
     access_ip: 192.168.8.88
   node4:
     ansible_host: 192.168.8.72
     ip: 192.168.8.72
     access_ip: 192.168.8.72
 children:
   kube-master:
     hosts:
       node1:
   kube-node:
     hosts:
       node1:
       node2:
       node3:
       node4:
   etcd:
     hosts:
       node1:
       node2:
       node3:
   k8s-cluster:
     children:
       kube-master:
       kube-node:
   calico-rr:
     hosts: {}
```
19. Modify the entries in offline.yaml changisng the ipaddress accordingly

`cat /root/kubespray-offline/kubespray/inventory/mycluster/group_vars/k8s-cluster/offline.yml`

```
---
## Global Offline settings
### Private Container Image Registry
registry_host: "192.168.8.191:5000"
files_repo: "http://192.168.8.191/packages"
### If using CentOS, RedHat or Fedora
# yum_repo: "http://myinternalyumrepo"
### If using Debian
#debian_repo: ""
### If using Ubuntu
ubuntu_repo: "http://192.168.8.191/packages/amd64/"

## Container Registry overrides
kube_image_repo: "{{ registry_host }}"
gcr_image_repo: "{{ registry_host }}"
docker_image_repo: "{{ registry_host }}"
quay_image_repo: "{{ registry_host }}"

## Kubernetes components
kubeadm_download_url: "{{ files_repo }}/kubernetes/kubeadm"
kubectl_download_url: "{{ files_repo }}/kubernetes/kubectl"
kubelet_download_url: "{{ files_repo }}/kubernetes/kubelet"

## CNI Plugins
cni_download_url: "{{ files_repo }}/kubernetes/cni-plugins-linux-{{ image_arch }}-{{ cni_version }}.tgz"

## cri-tools
# crictl_download_url: "{{ files_repo }}/kubernetes/cri-tools/crictl-{{ crictl_version }}-{{ ansible_system | lower }}-{{ image_arch }}.tar.gz"

## [Optional] etcd: only if you **DON'T** use etcd_deployment=host
# etcd_download_url: "{{ files_repo }}/kubernetes/etcd/etcd-{{ etcd_version }}-linux-amd64.tar.gz"

# [Optional] Calico: If using Calico network plugin
# calicoctl_download_url: "{{ files_repo }}/kubernetes/calico/{{ calico_ctl_version }}/calicoctl-linux-{{ image_arch }}"

## CentOS/Redhat
### For EL7, base and extras repo must be available, for EL8, baseos and appstream
### By default we enable those repo automatically
# rhel_enable_repos: false
### Docker / Containerd
# docker_rh_repo_base_url: "{{ yum_repo }}/docker-ce/$releasever/$basearch"
# docker_rh_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

## Fedora
### Docker
# docker_fedora_repo_base_url: "{{ yum_repo }}/docker-ce/{{ ansible_distribution_major_version }}/{{ ansible_architecture }}"
# docker_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"
### Containerd
# containerd_fedora_repo_base_url: "{{ yum_repo }}/containerd"
# containerd_fedora_repo_gpgkey: "{{ yum_repo }}/docker-ce/gpg"

## Debian
### Docker
# docker_debian_repo_base_url: "{{ debian_repo }}/docker-ce"
# docker_debian_repo_gpgkey: "{{ debian_repo }}/docker-ce/gpg"
### Containerd
# containerd_debian_repo_base_url: "{{ ubuntu_repo }}/containerd"
# containerd_debian_repo_gpgkey: "{{ ubuntu_repo }}/containerd/gpg"
# containerd_debian_repo_repokey: 'YOURREPOKEY'

## Ubuntu
### Docker
docker_ubuntu_repo_base_url: "{{ files_repo }}/docker-ce/"
docker_ubuntu_repo_gpgkey: "{{ files_repo }}/docker-ce/gpg"
### Containerd
# containerd_ubuntu_repo_base_url: "{{ ubuntu_repo }}/containerd"
# containerd_ubuntu_repo_gpgkey: "{{ ubuntu_repo }}/containerd/gpg"
# containerd_ubuntu_repo_repokey: 'YOURREPOKEY'

```
20. Run the Ansibe playbook now

`
 ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml -u ubuntu --private-key <path to the private key>
`

21. Validation whether the cluster is working fine

`
kubectl get nodes
`
#### Sample output
```
root@node1:~# kubectl get nodes
NAME    STATUS   ROLES                  AGE     VERSION
node1   Ready    control-plane,master   3h57m   v1.20.2
node2   Ready    <none>                 3h56m   v1.20.2
node3   Ready    <none>                 3h56m   v1.20.2
node4   Ready    <none>                 3h56m   v1.20.2
root@node1:~#
```

22. Validating whether the kubernetes cluster is using internal registry

`
kubectl deploy nginx --image=nginx
`

`
kubectl edit deploy nginx 
`
> Increase the replicas to 5s

```
root@node1:~# kubectl describe pod nginx-6799fc88d8-7p568
Name:         nginx-6799fc88d8-7p568
Namespace:    default
Priority:     0
Node:         node2/192.168.8.206
Start Time:   Sat, 30 Jan 2021 15:19:08 +0000
Labels:       app=nginx
              pod-template-hash=6799fc88d8
Annotations:  cni.projectcalico.org/podIP: 10.233.96.2/32
              cni.projectcalico.org/podIPs: 10.233.96.2/32
Status:       Running
IP:           10.233.96.2
IPs:
  IP:           10.233.96.2
Controlled By:  ReplicaSet/nginx-6799fc88d8
Containers:
  nginx:
    Container ID:   docker://a926cc1b14312c3e0cec22af576f3a5b1c0353b1c6d0ef38eb17e05a6ed757ae
    Image:          nginx
    Image ID:       docker-pullable://192.168.8.191:5000/library/nginx@sha256:0b159cd1ee1203dad901967ac55eee18c24da84ba3be384690304be93538bea8
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sat, 30 Jan 2021 15:19:16 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-j4wcj (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-j4wcj:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-j4wcj
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                 node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:          <none>
root@node1:~# 
```

