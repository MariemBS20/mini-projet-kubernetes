# mini-projet-kubernetes : Déploiement de WordPress avec Kubernetes (Manifests YAML & Kustomize)
## Contexte

Ce projet consiste à déployer une application WordPress dans un cluster Kubernetes en utilisant uniquement des manifests YAML, sans recourir à Helm.
L’objectif est de comprendre en profondeur les composants Kubernetes nécessaires à la mise en place d’une application complète :

- Base de données (MySQL)
- Frontend (WordPress)
  
et le rôle de chaque ressource Kubernetes (Deployment, Service, Volume, Secret, etc.).

## Mise en place de l’architecture Kubernetes

Dans le cadre de ce mini projet , l’objectif est de concevoir et déployer une architecture complète WordPress sur Kubernetes en utilisant exclusivement des manifests YAML.
Cette architecture repose sur la séparation des responsabilités entre la base de données (MySQL) et l’application frontend (WordPress), tout en appliquant les bonnes pratiques Kubernetes.

Les réalisations attendues sont les suivantes :

- Déployer l’application WordPress à l’aide de manifests YAML, sans utiliser Helm

- Mettre en place un Deployment MySQL avec un seul replica afin d’assurer la persistance de la base de données

- Exposer MySQL à l’intérieur du cluster via un Service de type ClusterIP

- Configurer un Deployment WordPress avec les variables d’environnement nécessaires pour la connexion à la base MySQL

- Assurer la persistance des données WordPress grâce à un volume monté dans le répertoire /data du nœud

- Rendre l’interface WordPress accessible depuis l’extérieur du cluster à l’aide d’un Service de type NodePort

##  Structure du projet

```text
kustomize-wordpress/
├── base/
│   ├── mysql/
│   │   ├── kustomization.yml
│   │   ├── mysql-pv.yml
│   │   ├── mysql-pvc.yml
│   │   ├── mysql-secret.yml
│   │   ├── mysql-deploy.yml
│   │   └── mysql-svc.yml
│   │
│   └── wordpress/
│       ├── wp-deploy.yml
│       ├── wp-secret.yml
│       └── wp-svc.yml
│
└── overlays/
    └── dev/
        └── kustomization.yml

```







- **base/** : ressources Kubernetes génériques et réutilisables  
- **overlays/** : adaptations spécifiques à l’environnement (ici : développement)

---

##  MySQL – Base de données

###  mysql-secret.yml

Stocke les informations sensibles :

- utilisateur MySQL  
- mot de passe  
- nom de la base de données  

Les valeurs sont encodées en **Base64**.

**Rôle :**
- Sécurité des identifiants  
- Partage des credentials avec WordPress  

---

### mysql-pv.yml – PersistentVolume

Définit un espace de stockage physique :

- Capacité : **10Gi**
- Stockage local (**hostPath**)
- Données conservées même après suppression du pod

**Rôle :**  
Garantir la persistance des données MySQL.

---

### mysql-pvc.yml – PersistentVolumeClaim

Représente une demande de stockage pour MySQL :

- Se lie automatiquement au PersistentVolume
- Abstrait le stockage pour le pod

**Rôle :**  
Permet à MySQL d’utiliser un volume persistant sans dépendre du stockage physique.

---

### mysql-deploy.yml – Deployment

Déploie MySQL dans Kubernetes :

- **1 replica**
- Image : `mysql:5.7`
- Variables d’environnement injectées depuis le Secret
- Volume monté sur `/var/lib/mysql`
- Limites CPU / mémoire définies

**Rôle :**  
Assurer la stabilité et la persistance de la base de données.

---

### mysql-svc.yml – Service

Expose MySQL à l’intérieur du cluster :

- Type : **ClusterIP**
- Port : **3306**



##  WordPress – Application Frontend

###  wp-secret.yml

Stocke les identifiants MySQL utilisés par WordPress :

- nom d’utilisateur  
- mot de passe  
- nom de la base de données  

**Rôle :**
- Sécuriser les informations sensibles  
- Réutiliser les mêmes credentials que MySQL afin d’assurer la cohérence de la connexion  

---

###  wp-deploy.yml – Deployment

Déploie l’application WordPress dans Kubernetes :

- Image : `wordpress:4.8-apache`
- Port exposé : **80**
- Connexion à MySQL via des variables d’environnement
- Volume monté sur `/data/wordpress` pour la persistance des fichiers
- Définition de limites CPU et mémoire

**Rôle :**  
Assurer la **résilience**, la **disponibilité** et la **persistance des données** WordPress.

---

###  wp-svc.yml – Service

Expose WordPress à l’extérieur du cluster Kubernetes :

- Type : **NodePort**
- Port externe : **30001**

**Accès à l’application :**

```text
http://<IP_du_nœud>:30001
```


##  Kustomize & Overlays

###  base

Contient les configurations communes et réutilisables de l’application :

- MySQL : **1 pod**
- WordPress : **1 pod**

Ces fichiers représentent la **configuration minimale fonctionnelle** permettant de déployer l’application WordPress avec sa base de données MySQL.

---

###  overlays/dev

Adapte la configuration pour l’environnement de développement **sans modifier les fichiers de base**.

```yaml
resources:
- ../../base/mysql
- ../../base/wordpress

patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: wp-deploy
  patch: |
    - op: replace
      path: /spec/replicas
      value: 3
```
 Dans l’environnement **dev**, WordPress est déployé avec **3 replicas**, ce qui permet :

- une meilleure **disponibilité** de l’application en cas de défaillance d’un pod,  
- une **répartition de la charge** entre plusieurs pods WordPress,  
- une **meilleure tolérance aux pannes**,  
- **sans duplication ni modification** des manifests de base grâce à Kustomize.

---

##  Déploiement de l’application

Commande de déploiement :

```bash
kubectl apply -k overlays/dev
```
##  Vérification de l’état du cluster

Après le déploiement, il est possible de vérifier l’état des ressources Kubernetes à l’aide des commandes suivantes :

```bash
kubectl get pods
kubectl get svc
```


