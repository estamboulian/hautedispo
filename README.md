# Rapport de Projet : Projet Installation H/A d'un Cluster Kubernetes

## 1. Architecture du Projet
```
Haute Dispo/
├── HostPath
│   ├── 01-storage.yaml
│   ├── 02-mysql.yaml
│   └── 03-wordpress.yaml
└── NFS
    ├── mysql
    │   ├── mysql1.yaml
    │   ├── mysql-final.yaml
    │   ├── mysql-sc.yaml
    │   ├── mysql-svc.yaml
    │   └── mysql.yaml
    ├── nfs-sc.yml
    ├── storage
    │   ├── pvc.yaml
    │   ├── pv.yaml
    │   └── storage.yaml
    └── wordpress
        ├── wordpress-final.yaml
        ├── wp-svc.yaml
        └── wp.yaml

6 directories, 15 files
```
### Topologie Réseau
* 3 Control-Plane :
  - CP 1 : 10.1.10.66
  - CP 2 : 10.1.10.67
  - CP 3 : 10.1.10.69

* 2 Workers :
  - Worker 1 : 10.1.10.65
  - Worker 2 : 10.1.10.68

* HA Proxy :
  - IP : 10.1.10.63
  - VIP : 10.1.10.7

### Schéma Réseau de notre Infrastructure
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

* **Mise en place du stockage NFS**

Le stockage de l’application repose sur un serveur NFS centralisé, qui expose des répertoires partagés accessibles depuis l’ensemble des nœuds du cluster Kubernetes (Control Plane et Workers).

Sur le serveur NFS, une arborescence dédiée à Kubernetes est créée, par exemple :

    * /srv/nfs/k8s/mysql : stockage des données MySQL

    * /srv/nfs/k8s/wordpress : stockage des fichiers WordPress (thèmes, plugins, médias)

Ces répertoires sont exportés via NFS avec des droits permettant la lecture et l’écriture depuis les nœuds du cluster. Chaque Pod peut ainsi monter ces dossiers comme s’il s’agissait d’un disque local.

* **PersistentVolumes (PV)**
* Les PersistentVolumes définissent le lien entre Kubernetes et le serveur NFS :
* Chaque PV référence :
    * l’adresse IP du serveur NFS,
    * le chemin exact du répertoire exporté,
    * le type de stockage (nfs),
    * la capacité logique allouée.

Le mode d’accès ReadWriteMany (RWX) est utilisé, ce qui permet à plusieurs Pods de monter simultanément le même volume, un point essentiel pour WordPress en haute disponibilité.
Les PV agissent comme une abstraction du stockage physique, évitant toute dépendance à un nœud spécifique.

* **PersistentVolumeClaims (PVC)**

Les PersistentVolumeClaims permettent aux Pods de demander un espace de stockage sans connaître les détails du serveur NFS :

Les PVC réclament un volume compatible en termes de :
    * capacité,
    * mode d’accès (RWX),
    * classe de stockage.
* Kubernetes associe automatiquement chaque PVC au PV correspondant.

Cette approche rend le déploiement modulaire et réutilisable, tout en simplifiant la gestion du stockage.

* **Persistance et cohérence des données**

Grâce au NFS :
    * Les données MySQL restent disponibles même après un redémarrage du Pod.
    * Les fichiers WordPress sont partagés entre toutes les instances de l’application.
    * Les mises à jour (upload de médias, installation de plugins) sont immédiatement visibles sur tous les Pods WordPress.
* Cette architecture garantit la continuité du service, la cohérence des données et facilite la montée en charge de l’application.

* **Mysql**
* **Déploiement MySQL :**
    * MySQL est déployé avec un **seul réplica**, afin d’éviter les conflits d’accès disque (notamment les verrous InnoDB) sur un stockage NFS partagé.

* **Stockage persistant :**
    * Les données de la base sont stockées sur un volume NFS dédié, assurant leur conservation indépendamment du cycle de vie du Pod.

* **Variables d’environnement :**
    * L’initialisation automatique de la base de données (nom de la base, utilisateur, mot de passe) est gérée via des variables d’environnement.

* **Service interne :**
    * Un service Kubernetes de type ClusterIP fournit un point d’accès stable à MySQL, permettant à WordPress de s’y connecter sans dépendre de l’adresse IP du Pod

* **Gestion des secrets MySQL :**
    * Les informations **sensibles** de la base de données (mot de passe root, utilisateur et mot de passe applicatif) sont stockées dans des Secrets Kubernetes. Cette approche garantit la protection des identifiants, évite leur exposition en clair dans les fichiers de      déploiement et permet une configuration sécurisée et centralisée des Pods MySQL et WordPress.

* **Wordpress**
* **Déploiement WordPress :**
    * WordPress est déployé avec **plusieurs réplicas**, ce qui permet d’assurer une haute disponibilité de l’application.

* **Stockage partagé via NFS :**
    * Les fichiers WordPress sont montés depuis un volume NFS commun, garantissant que tous les Pods utilisent exactement les mêmes données (extensions, thèmes, fichiers médias).

* **Exposition du service :**
    * L’application est rendue accessible depuis l’extérieur du cluster à l’aide d’un service de type NodePort, permettant aux utilisateurs d’accéder au site web via un port fixe.

###  Déploiement du Wordpress avec HostPath
* **Transition vers le HostPath :**
    * **Stabilité accrue :** Le choix du **HostPath** a été privilégié pour garantir une performance d'accès direct au disque (I/O) et éliminer les erreurs de montage réseau.

* **01-storage.yaml (La couche de persistance) :**
    * **PersistentVolumes (PV) :** Définition des volumes physiques pointant vers les dossiers locaux sécurisés du nœud.
    * **PersistentVolumeClaims (PVC) :** Création des "bons de commande" qui permettent aux Pods de réclamer leur espace de stockage.
    * **Indépendance des données :** Cette structure garantit que les fichiers WordPress et les tables MySQL survivent même si les Pods sont supprimés ou déplacés.

* **02-mysql.yaml (Le socle de données) :**
    * **Déploiement :** Gestion du moteur MySQL 8.0 avec une configuration optimisée pour la persistance.
    * **Variables d'environnement :** Automatisation de l'initialisation de la base de données (nom, utilisateur et mot de passe).
    * **Service :** Mise en place d'un point d'accès interne stable pour que WordPress puisse contacter la base sans se soucier de l'IP changeante du Pod.

* **03-wordpress.yaml (Le serveur applicatif) :**
    * **Haute Disponibilité :** Configuration de **2 réplicas** pour assurer la continuité de service en cas de défaillance d'une instance.
    * **Exposition :** Utilisation d'un service **NodePort** sur le port **31803**, rendant le site accessible depuis l'extérieur.
    * **Liaison (Labels) :** Utilisation de sélecteurs d'étiquettes pour créer un écosystème cohérent où le stockage, la base de données et le serveur web se reconnaissent mutuellement.
---

## 5. Gestion du Cycle de Vie : Procédure de Mise à Jour
La mise à jour du cluster (via `kubeadm`) est une opération critique réalisée selon une méthodologie "Node-by-Node" pour garantir la disponibilité du site.

### 5.1 Mesures de Sécurité (Backup)
Avant toute action, nous procédons à une sauvegarde du cerveau du cluster (Etcd) :
```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db \
  --endpoints=[https://127.0.0.1:2379](https://127.0.0.1:2379) \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 5.2 Séquence de Mise à Jour
1. **Control Plane :** Mise à jour séquentielle des Masters. On applique `kubeadm upgrade apply` sur le premier, puis `upgrade node` sur les suivants.
2. **Workers :**  **Drainage** : `kubectl drain <node>` pour déplacer les Pods en douceur.
* **Update :** Mise à jour des paquets `kubeadm`, `kubelet` et `kubectl`.
* **Réintégration :** `kubectl uncordon` pour remettre le nœud en service.

### 5.3 Validation Post-Update

* Vérification que tous les nœuds sont en `Ready` via `kubectl get nodes`.
* Test d'accès final sur l'URL du service pour confirmer l'intégrité du site WordPress.

---
## 6. Auteurs
* **Édouard Stamboulian** — [GitHub de Édouard](https://github.com/estamboulian)  
* **Ahmed Khairi** — [GitHub de Ahmed](https://github.com/hirlho)  


