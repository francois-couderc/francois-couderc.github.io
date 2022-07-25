# Fondamentaux kubernetes et calico

## Création des nodes

#### Prérequis

image : ubuntu-22.04-live-server-amd64.iso

CPUs : 4 / RAM : 32G / Disk  : 60G Disk thin provisioning

#### IP & credentials

hostname : **node-1**  / interface ens160 : 10.0.0.1/24 (IPDG : .254)

```
root@node-1:~# cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens160:
      addresses:
      - 10.0.0.1/24
      gateway4: 10.0.0.254
      nameservers:
        addresses:
        - 192.168.123.5
        search: []
  version: 2
```

```
sudo netplan apply
```

hostname : **node-2** / interface ens160 : 10.0.0.2/24 (IPDG : .254)

```
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens160:
      addresses:
      - 10.0.0.2/24
      gateway4: 10.0.0.254
      nameservers:
        addresses:
        - 192.168.123.5
        search: []
  version: 2
```

```
sudo netplan apply
```

Les deux VMs sont sur le même ESX. Les deux interfaces ens160 (underlay) se partagent le même port-group.

Credentials pour les 2 VMs : labuser/cisco123

#### Accès route en SSH

> Optionnel, permet d’éditer les fichiers de configuration (nécessitant sudo) de la VM TIG à distance en sftp.

Dans le fichier `/etc/ssh/sshd_config`, remplacer

```
#PermitRootLogin prohibit-password
```

par

```
PermitRootLogin yes
```

Redémarrer le service SSH :

```
sudo systemctl restart ssh
```

Assigner un mot de passe au compte root :

```
labuser@node-1:~$ sudo passwd
New password:
Retype new password:
passwd: password updated successfully
```

#### Configuration NTP client

Ajouter l’IP du serveur NTP dans `/etc/systemd/timesyncd.conf` :

```
[Time]
NTP=192.168.123.254
```

Redémarrer le service NTP :

```
sudo systemctl restart systemd-timesyncd.service
```

Changer la timezone à Paris (CET +1) :

```
sudo timedatectl set-timezone Europe/Paris
```

Vérification de la synchronisation NTP :

```
labuser@telemetry:~$ timedatectl
               Local time: Mon 2022-03-21 09:25:33 CET
           Universal time: Mon 2022-03-21 08:25:33 UTC
                 RTC time: Mon 2022-03-21 08:25:34
                Time zone: Europe/Paris (CET, +0100)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

#### Désactiver le swap sur les VMs

Modifier le fichier `/etc/fstab`:

```
#/dev/mapper/VM--Template--vg-swap_1 none            swap    sw              0       0
```

puis

```
sudo swapoff -a
```

## Installation de CRI-O et kubernetes

> A partir de cette étape, les noeuds sont accédas en SSH avec le compte root

### Configuration réseau des nodes

Création du fichier de configuration pour charger les modules réseau `overlay` et `br_filter` au boot :

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

Activation des modules réseaux :

```
modprobe overlay
modprobe br_netfilter
```

Kubernetes nécessite que les packets traversant une network bridge soit processés pour du filtrage et du port forwarding via iptables. Pour ce faire, il faut configurer les modules réseau du kernel :

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

Prise en compte des changements de configurations :

```
sysctl --system
```

### Installation de CRI-O

CRI-O est une implémentation de l’interface d’environnement d’exécution de conteneurs (Container Runtime Interface, CRI) pour [Kubernetes](https://www.ionos.fr/digitalguide/serveur/know-how/kubernetes/), qui utilise les images et les environnements d’exécution (runtimes) de « l’Open Container Initiative » (OCI). C'est une alternative à docker (dont le runtime natif n'est plus supporté et est remplacé par `containerd.io`)

Configuration des variables d'environnements qui vont être utilisées pour l'installation :

```
OS=xUbuntu_22.04
VERSION=1.24
```

Ajout du repository Kubic :

```
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" | \
tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
```

Import de la clé publique associée :

```
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | \
apt-key add -
```

Ajout du repository CRI associé :

```
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" | \
tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
```

Import de la clé publique associée :

```
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | \
apt-key add -
```

Installation des packages CRI :

```
apt update
apt install cri-o cri-o-runc cri-tools -y
```

Activation des packages :

```
systemctl enable crio.service
systemctl start crio.service
```

Vérification du bon fonctionnement de CRI avec la commande de gestion CLI `crictl`:

```
root@node-1:~# crictl info
{
  "status": {
    "conditions": [
      {
        "type": "RuntimeReady",
        "status": true,
        "reason": "",
        "message": ""
      },
      {
        "type": "NetworkReady",
        "status": false,
        "reason": "NetworkPluginNotReady",
        "message": "Network plugin returns error: No CNI configuration file in /etc/cni/net.d/. Has your network provider started?"
      }
    ]
  }
}
```

### Installation de kubernetes

Ajout du repository kubernetes :

```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
```

Import de la clé publique associée :

```
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

Installation des packages kubernetes :

```
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
```

`kubeadm` : outil d'installation d'un cluster kubernetes

`kubelet` :  agent qui s'exécute sur chaque nœud du cluster. Il s'assure que les conteneurs fonctionnent dans un pod.

`kube-proxy` : proxy réseau qui s'exécute sur chaque nœud du cluster et implémente une partie du concept Kubernetes de Service. kube-proxy maintient les règles réseau sur les nœuds. Ces règles réseau permettent une communication réseau vers les Pods depuis des sessions réseau à l'intérieur ou à l'extérieur du cluster. kube-proxy utilise la couche de filtrage de paquets du système d'exploitation.

## Configuration du cluster

#### Mise en route du cluster kubernetes

> La configuration du cluster se fait depuis le noeud master uniquement. Node-1 dans ce setup.

Récupération de manière proactive des images qui vont être utilisées pour l'installation du cluster :

```
root@node-1:~# kubeadm config images pull
[config/images] Pulled k8s.gcr.io/kube-apiserver:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-controller-manager:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-scheduler:v1.24.3
[config/images] Pulled k8s.gcr.io/kube-proxy:v1.24.3
[config/images] Pulled k8s.gcr.io/pause:3.7
[config/images] Pulled k8s.gcr.io/etcd:3.5.3-0
[config/images] Pulled k8s.gcr.io/coredns/coredns:v1.8.6kubeadm config images pull
```

Initialisation du cluster avec comme cidr pour les pods 100.0.0.0/16 pour les IPs de pods:

```
root@node-1:~# kubeadm init --pod-network-cidr=100.0.0.0/16  --token-ttl 0
[init] Using Kubernetes version: v1.24.3

<SNIP>

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.0.0.1:6443 --token t5hgnc.85i4m5qlfuq7hza6 \
	--discovery-token-ca-cert-hash sha256:4eb04b6ddc5dfe19f17b6c0980cb5300ef877c015b0560d39b7f1f43b22c4d14
```

Toujours depuis node-1, prise en compte du fichier de config kubernetes par défaut :

```
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Configuration de la complétion de commande kubernetes :

```
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

A ce stade, la configuration du cluster est la suivante. Tous les services de sont pas encore montés, notamment parce que le CNI n'est pas encore installé :

```
root@node-1:~# kubectl get pods --all-namespaces -o wide
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE     IP         NODE     NOMINATED NODE   READINESS GATES
kube-system   coredns-6d4b75cb6d-6r8q5         0/1     Pending   0          8m36s   <none>     <none>   <none>           <none>
kube-system   coredns-6d4b75cb6d-qgv4p         0/1     Pending   0          8m36s   <none>     <none>   <none>           <none>
kube-system   etcd-node-1                      1/1     Running   0          8m48s   10.0.0.1   node-1   <none>           <none>
kube-system   kube-apiserver-node-1            1/1     Running   0          8m51s   10.0.0.1   node-1   <none>           <none>
kube-system   kube-controller-manager-node-1   1/1     Running   0          8m49s   10.0.0.1   node-1   <none>           <none>
kube-system   kube-proxy-sb2g2                 1/1     Running   0          8m36s   10.0.0.1   node-1   <none>           <none>
kube-system   kube-scheduler-node-1            1/1     Running   0          8m49s   10.0.0.1   node-1   <none>           <none>
```

#### Installation du CNI calico

> Basé sur le documentation d'installation de Tigera : https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises

Installation de l'operator sur le cluster :

```
kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
```

Téléchargement du fichier de configuration pour personnalisé le setup (CIDR range, taille du subnet pour chaque node) :

```
curl https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml -O
```

Personnalisation du fichier ( CIDR, ...). A ce stade, VXLAN est configuré, nous allons le désactiver plus tard :

```
root@node-1:~# cat custom-resources.yaml
# This section includes base Calico installation configuration.
# For more information, see: https://projectcalico.docs.tigera.io/v3.23/reference/installation/api#operator.tigera.io/v1.Installation
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  # Configures Calico networking.
  calicoNetwork:
    # Note: The ipPools section cannot be modified post-install.
    ipPools:
    - blockSize: 24
      cidr: 100.0.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
---
# This section configures the Calico API server.
# For more information, see: https://projectcalico.docs.tigera.io/v3.23/reference/installation/api#operator.tigera.io/v1.APIServer
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
```

Création du manifest pour installer calico :

```
kubectl create -f custom-resources.yaml
```

L'état du cluster doit etre maintenant le suivant :

```
root@node-1:~# kubectl get pods --all-namespaces -o wide
NAMESPACE         NAME                                      READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
calico-system     calico-kube-controllers-657d56796-g8962   0/1     Pending   0          59s     <none>       <none>   <none>           <none>
calico-system     calico-node-k5t5r                         1/1     Running   0          59s     10.0.0.1     node-1   <none>           <none>
calico-system     calico-typha-66f485fdc9-4mdfs             1/1     Running   0          59s     10.0.0.1     node-1   <none>           <none>
kube-system       coredns-6d4b75cb6d-65h45                  1/1     Running   0          10m     100.0.25.2   node-1   <none>           <none>
kube-system       coredns-6d4b75cb6d-kd77j                  1/1     Running   0          10m     100.0.25.1   node-1   <none>           <none>
kube-system       etcd-node-1                               1/1     Running   0          10m     10.0.0.1     node-1   <none>           <none>
kube-system       kube-apiserver-node-1                     1/1     Running   0          10m     10.0.0.1     node-1   <none>           <none>
kube-system       kube-controller-manager-node-1            1/1     Running   0          10m     10.0.0.1     node-1   <none>           <none>
kube-system       kube-proxy-kzgq2                          1/1     Running   0          10m     10.0.0.1     node-1   <none>           <none>
kube-system       kube-scheduler-node-1                     1/1     Running   0          10m     10.0.0.1     node-1   <none>           <none>
tigera-operator   tigera-operator-6995cc5df5-cgmbx          1/1     Running   0          9m57s   10.0.0.1     node-1   <none>           <none>
```

Le pod `calico-kube-controllers` est encore à l'état pending. Cela vient du fait que node-1 est actuellement l'unique noeud du cluster et qu'il est configuré par défaut pour ne pas accueillir de pods. 

La commande suivant permet au node node-1 de fonctionner également comme un worker :

```
kubectl taint node node-1 node-role.kubernetes.io/control-plane:NoSchedule-
```

L'ensemble des pods calico est alors opérationnel :

```
root@node-1:~# kubectl get pods --all-namespaces -o wide
NAMESPACE          NAME                                      READY   STATUS    RESTARTS   AGE     IP           NODE     NOMINATED NODE   READINESS GATES
calico-apiserver   calico-apiserver-654ddbb5b7-8q4q4         0/1     Running   0          18s     100.0.25.4   node-1   <none>           <none>
calico-apiserver   calico-apiserver-654ddbb5b7-ncm6p         0/1     Running   0          18s     100.0.25.5   node-1   <none>           <none>
calico-system      calico-kube-controllers-657d56796-g8962   1/1     Running   0          4m50s   100.0.25.3   node-1   <none>           <none>
calico-system      calico-node-k5t5r                         1/1     Running   0          4m50s   10.0.0.1     node-1   <none>           <none>
calico-system      calico-typha-66f485fdc9-4mdfs             1/1     Running   0          4m50s   10.0.0.1     node-1   <none>           <none>
kube-system        coredns-6d4b75cb6d-65h45                  1/1     Running   0          14m     100.0.25.2   node-1   <none>           <none>
kube-system        coredns-6d4b75cb6d-kd77j                  1/1     Running   0          14m     100.0.25.1   node-1   <none>           <none>
kube-system        etcd-node-1                               1/1     Running   0          14m     10.0.0.1     node-1   <none>           <none>
kube-system        kube-apiserver-node-1                     1/1     Running   0          14m     10.0.0.1     node-1   <none>           <none>
kube-system        kube-controller-manager-node-1            1/1     Running   0          14m     10.0.0.1     node-1   <none>           <none>
kube-system        kube-proxy-kzgq2                          1/1     Running   0          14m     10.0.0.1     node-1   <none>           <none>
kube-system        kube-scheduler-node-1                     1/1     Running   0          14m     10.0.0.1     node-1   <none>           <none>
tigera-operator    tigera-operator-6995cc5df5-cgmbx          1/1     Running   0          13m     10.0.0.1     node-1   <none>           <none>
```

#### Ajout du deuxième noeud worker node-2

Sur node-2, lancer la commande affichée lors du lancer de kubeadm sur node-1. Cette commande permet au node-2 de s'insérer dans le cluster en tant que worker :

```
kubeadm join 10.0.0.1:6443 --token t5hgnc.85i4m5qlfuq7hza6 \
	--discovery-token-ca-cert-hash sha256:4eb04b6ddc5dfe19f17b6c0980cb5300ef877c015b0560d39b7f1f43b22c4d14
root@node-2:~# kubeadm join 10.0.0.1:6443 --token t5hgnc.85i4m5qlfuq7hza6 \
        --discovery-token-ca-cert-hash sha256:4eb04b6ddc5dfe19f17b6c0980cb5300ef877c015b0560d39b7f1f43b22c4d14
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

Vérification depuis node-1 (master) :

```
root@node-1:~# kubectl get nodes
NAME     STATUS   ROLES           AGE     VERSION
node-1   Ready    control-plane   33m     v1.24.3
node-2   Ready    <none>          2m19s   v1.24.3
```

Vérification de la création des pods de services sur node-2 :

```
root@node-1:~# kubectl get pods --all-namespaces -o wide | grep node-2
calico-system      calico-node-z9r5d                         1/1     Running   0          8m13s   10.0.0.2     node-2   <none>           <none>
kube-system        kube-proxy-cx299                          1/1     Running   0          8m13s   10.0.0.2     node-2   <none>           <none>
```

#### Installation de calicoctl

`calicoctl` est l'outil CLI qui permet de commander calico. Il doit être installé sur les tous les noeuds master (node-1.

Pour l'installation, il suffit d'installer le binaire dans un répertoire qui est dans la variable d'environnement `PATH` :

```
cd /usr/local/bin
curl -L https://github.com/projectcalico/calico/releases/download/v3.23.3/calicoctl-linux-amd64 -o calicoctl
chmod +x ./calicoctl
cd
```

Vérification sur node-1 :

```
root@node-1:~# calicoctl version
Client Version:    v3.23.3
Git commit:        3a3559be1
Cluster Version:   v3.23.3
Cluster Type:      typha,kdd,k8s,operator,bgp,kubeadm
```

La commande suivante permet de voit le que le peering BGP entre node-1 et node-2 est bien dans l'état established :

```
root@node-1:~# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.0.2     | node-to-node mesh | up    | 14:17:30 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

#### Tweak pour utiliser `ping` dans les pods

Par défaut CRI-O interdit le lancement de ping depuis les PODs pour des raisons de sécurité. Cette protection doit être désactiver sur tous les noeuds (qui hostent des pods).

 Modifier le fichier `/etc/crio/crio.conf` et rajouter le champ `NET_RAW` dans les `default_capabilities` :

```
 default_capabilities = [
        "CHOWN",
        "DAC_OVERRIDE",
        "FSETID",
        "FOWNER",
        "SETGID",
        "SETUID",
        "SETPCAP",
        "NET_BIND_SERVICE",
        "KILL",
        "NET_RAW",
 ]
```

Puis redémarrer  CRI-O :

```
systemctl restart crio
```













## Aide-mémoire kubectl

https://kubernetes.io/fr/docs/reference/kubectl/cheatsheet/

