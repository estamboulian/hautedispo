# Rapport de Projet : Infrastructure Kubernetes Haute Disponibilité (HA)

## 1. Architecture du Projet


### Schéma Réseaux de notre Infrastructure
[Image]
---

## 2. Architecture Réseau & Haute Disponibilité

### Équilibrage de Charge (Load Balancing)
L'accès à l'API Kubernetes est sécurisé par un couple **HAProxy** et **Keepalived**. Une adresse IP Virtuelle (VIP) `10.1.10.7` sert de point d'entrée unique.

* **HAProxy** : Répartit les requêtes sur les trois nœuds du Control Plane (port 6443) en mode `round-robin`.
* **Keepalived** : Assure la haute disponibilité du Load Balancer. En cas de panne du nœud principal, l'IP virtuelle bascule instantanément sur un nœud secondaire.

### Détails de la Configuration (Extraits)
**HAProxy :**
```haproxy
frontend kubernetes
    bind *:6443
    mode tcp
    default_backend k8s-masters

backend k8s-masters
    balance roundrobin
    server cp1 10.1.10.66:6443 check
    server cp2 10.1.10.67:6443 check
    server cp3 10.1.10.69:6443 check
```
**Keepalived :**
```keepalived
vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 51
    priority 101
    virtual_ipaddress {
        10.1.10.7
    }
}
```

---

## 3. Stratégie de Stockage Persistant (NFS)
Pour permettre aux pods de se déplacer librement entre les nœuds tout en conservant leurs données, nous avons implémenté une solution de stockage NFS (Network File System).
* Centralisation : Les fichiers /var/www/html de WordPress et les données /var/lib/mysql résident sur un serveur de stockage dédié.
* PersistentVolumes (PV) : Des volumes de 5Go ont été créés pour mapper les partages NFS au sein du cluster.
* Multi-Read/Write : La configuration NFS permet aux multiples réplicas de WordPress d'accéder simultanément aux mêmes ressources médiatiques.

---

## 4. Déploiement de l'Application
### Déploiement du Wordpress avec NFS

###  Déploiement du Wordpress avec HostPath
Après avoir testé la solution NFS, nous avons rapidement réalisé que dans un environnement de TP ou sur des serveurs aux performances d'écriture limitées, le stockage réseau pouvait devenir un goulot d'étranglement. Nous avons donc basculé sur une stratégie en HostPath.

L'ensemble de notre déploiement s'articule autour de trois fichiers YAML structurés, qui forment la colonne vertébrale de l'infrastructure. Le premier, 01-storage.yaml, définit la couche de persistance : il déclare les PersistentVolumes (PV) qui pointent vers nos dossiers locaux et les PersistentVolumeClaims (PVC) qui agissent comme des "bons de commande" pour l'application. Cette séparation permet de détacher l'application du stockage physique ; même si le Pod est supprimé, le PVC conserve le lien vers les données sur le disque, garantissant qu'aucune configuration WordPress ou table MySQL ne soit perdue.

Viennent ensuite les fichiers 02-mysql.yaml et 03-wordpress.yaml, qui gèrent respectivement la base de données et le serveur web. Chaque fichier combine un Service pour la communication réseau et un Deployment pour la gestion des containers. Dans le fichier MySQL, nous avons configuré les variables d'environnement pour l'initialisation de la base, tandis que le fichier WordPress définit nos deux réplicas et l'exposition en NodePort sur le port 31803. Le lien entre ces trois fichiers est assuré par les sélecteurs de labels, créant un écosystème où chaque composant se reconnaît et communique automatiquement dès son apparition dans le cluster.

