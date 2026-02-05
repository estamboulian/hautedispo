Conf haproxy:

global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # See: https://ssl-config.mozilla.org/#server=haproxy&server-version=2.0.3&config=intermediate
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http



frontend kubernetes
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-masters

backend k8s-masters
    mode tcp
    balance roundrobin
    option tcp-check
    server cp1 10.1.10.66:6443 check
    server cp2 10.1.10.67:6443 check
    server cp3 10.1.10.69:6443 check





Keepalive:

# Script pour vérifier si HAProxy est vivant
vrrp_script chk_haproxy {
    script "killall -0 haproxy" # Vérifie si le processus existe
    interval 2                  # Vérifie toutes les 2 secondes
    weight 2                    # Ajoute 2 points de priorité si OK
    user root
}

# Configuration de l'instance VRRP
vrrp_instance VI_1 {
    state MASTER           # "MASTER" sur le serveur principal
    interface eth0         # <--- CORRECTION ICI (c'est eth0, pas if35)
    virtual_router_id 51   # ID unique pour ce groupe VRRP
    priority 101           # 101 pour le Master
    advert_int 1

    authentication {
        auth_type PASS
        auth_pass esgi
    }

    virtual_ipaddress {
        10.1.10.7      # Votre VIP
    }

    track_script {
        chk_haproxy
    }
}


Architecture:

1 Haproxy, 3 controle plane et 2 worker

Installer de wordpress avec mysql
Partage NFS



Process de mise à jour:

🔹 Mise à jour d’un cluster Kubernetes HA (kubeadm)
1️⃣ Pré-requis

Avant toute mise à jour :

Sauvegarde du cluster et des applications

Sauvegarde etcd (control-plane)

ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key


Vérifie que tes manifests, ConfigMaps, Secrets, PV/PVC sont sauvegardés

Sauvegarde Longhorn ou autre stockage externe si nécessaire

Vérifier que tous les nœuds sont Ready

kubectl get nodes


Tous doivent être Ready.

Installer la nouvelle version de kubeadm sur tous les nœuds

sudo apt update
sudo apt install -y kubeadm=<nouvelle-version>


Ex: pour passer à 1.34.5 :

sudo apt install -y kubeadm=1.34.5-00


Mettre à jour le control-plane principal (un nœud à la fois)

### Planification

kubeadm upgrade plan


Vérifie la version actuelle et la version disponible

Note la version cible

Appliquer la mise à jour kubeadm sur le premier control-plane

sudo kubeadm upgrade apply v1.34.5

kubeadm met à jour les manifests et les composants control-plane (API server, controller-manager, scheduler)

### Mettre à jour kubelet et kubectl

sudo apt install -y kubelet=1.34.5-00 kubectl=1.34.5-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet


### Vérifier le nœud

kubectl get nodes
kubectl get pods -n kube-system


Tous les pods control-plane doivent être Running.

### Mettre à jour les autres control-plane

Répéter l’étape 2 un nœud à la fois.

Kubernetes HA permet de garder le cluster opérationnel pendant la mise à jour d’un nœud control-plane.

### Mettre à jour les workers

Installer kubeadm, kubelet et kubectl sur chaque worker

sudo apt install -y kubeadm=1.34.5-00 kubelet=1.34.5-00 kubectl=1.34.5-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet


Vérifier que le worker rejoint bien le cluster

kubectl get nodes

### Verifier le cluster après mise à jour

Tous les nœuds doivent être Ready

Tous les pods du namespace kube-system doivent être Running

Les applications Stateful (Longhorn, StatefulSets) doivent être opérationnelles

Vérifier la version

kubectl version --short

### Post-mise à jour

Vérifier le StorageClass et les PV/PVC Longhorn

Tester un déploiement test (ex: StatefulSet 2 réplicas)

Vérifier les endpoints HAProxy si exposés


Mettre à jour un nœud à la fois, surtout pour HA

Ne pas mettre à jour kubeadm après kubelet → kubelet doit rester compatible

Faire la mise à jour hors production si possible, ou avec un drain planifié

kubectl drain <node-name> --ignore-daemonsets


puis après update :

kubectl uncordon <node-name>


Garder un snapshot etcd récent avant chaque upgrade

