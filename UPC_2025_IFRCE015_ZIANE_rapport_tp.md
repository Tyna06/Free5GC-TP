# Université Paris Cité  
## Master Informatique – Réseaux et Systèmes Autonomes  
### Année universitaire 2024–2025  

---

# **Rapport de TP : Déploiement de Free5GC sur Kubernetes (KinD)**  
Module : **IFRCE015 – Réseaux 5G et Virtualisation**

---

## **Étudiante**  
- **Ziane Thinhinane**  

## **Enseignant**  
[Nom du professeur]

---

*(Ce document présente le travail réalisé dans le cadre du TP portant sur l’installation et le déploiement d’un cœur 5G open source Free5GC dans un environnement Kubernetes basé sur KinD.)*

---

## Table des matières
- [Introduction](#introduction)
- [1. Installation de la machine virtuelle](#1-installation-de-la-machine-virtuelle)
  - [1.1 Création de la machine virtuelle](#11-création-de-la-machine-virtuelle)
  - [1.2 Installation du système Ubuntu](#12-installation-du-système-ubuntu)
  - [1.3 Configuration de l’accès SSH](#13-configuration-de-laccès-ssh)
- [2. Installation des dépendances](#2-installation-des-dépendances)
- [3. Création du cluster KinD](#3-création-du-cluster-kind)
- [4. Installation du CNI et de Multus](#4-installation-du-cni-et-de-multus)
- [5. Configuration Free5GC](#5-configuration-free5gc)
- [6. Déploiement Free5GC](#6-déploiement-free5gc)
- [7. Déploiement UERANSIM](#7-déploiement-ueransim)
- [8. Problèmes rencontrés](#8-problèmes-rencontrés)
- [Conclusion](#conclusion)

---

# Introduction

Ce rapport présente l’ensemble des étapes nécessaires pour installer et déployer **Free5GC**, un cœur 5G open-source, sur un cluster Kubernetes basé sur **KinD**.  
L’objectif principal de ce TP est de comprendre :

- la mise en place d’un environnement virtualisé ;
- l’installation de Docker, Kind, Kubectl et Helm ;
- la gestion des réseaux CNI et l’ajout de Multus ;
- le déploiement complet de Free5GC ;
- l’exécution d’UE simulés via UERANSIM ;
- ainsi que l’analyse des problèmes rencontrés et leurs solutions.

Toutes les étapes sont illustrées par des captures d’écran et accompagnées de commentaires détaillés afin de fournir une documentation claire et pédagogique.

---

# 1. Installation de la machine virtuelle

Pour commencer ce TP, j’ai téléchargé l’image ISO de **Ubuntu Server 24.04 LTS** depuis le site officiel.  
Cette version est légère, stable et compatible avec les outils nécessaires au déploiement de Free5GC.

La première étape a consisté à créer une **machine virtuelle (VM)** à partir de cette image ISO.  
Pour cela, j’ai utilisé **VMware Workstation**, en sélectionnant l’option *Create a New Virtual Machine*, puis en important l’image ISO précédemment téléchargée.

Les sous-sections suivantes décrivent en détail toutes les étapes de création et configuration de cette VM.

---

## 1.1 Création de la machine virtuelle

![Création de la VM](./instalations_vm/01_vm_creation_start.png)

![Création – Étape suivante](./instalations_vm/01_01_vm_creation_start.png)

![Sélection de l’ISO](./instalations_vm/02_select_iso.png)
![Choix du nom et de l’emplacement](./instalations_vm/01_01_02vm_name_selection.png)


![Configuration CPU/RAM](./instalations_vm/03_vm_resources.png)

Dans cette étape, j’ai augmenté la taille du disque à **50 Go** afin de prévoir suffisamment d’espace pour les images Docker, Kubernetes et Free5GC.

![Configuration du disque](./instalations_vm/04_vm_disk.png)

J’ai également configuré **6 Go de RAM**, ce qui est recommandé pour ce TP en raison des ressources nécessaires pour faire tourner KinD et Free5GC.

![Premier boot Ubuntu](./instalations_vm/05_ubuntu_boot.png)

Cette capture montre la fin de la configuration et le lancement du système depuis l’ISO Ubuntu Server.

---

## 1.2 Installation du système Ubuntu

Après le démarrage sur l’image ISO, j’ai suivi les étapes d’installation de **Ubuntu Server**, notamment :

- choix de la langue et du clavier ;
- configuration automatique du réseau ;
- installation standard du système ;
- activation de l’option **OpenSSH Server** pour pouvoir utiliser SSH ;
- création de l’utilisateur administrateur ;
- test de connexion après installation.

Voici les captures d’écran illustrant le déroulement complet de l’installation :

![Installation Ubuntu – écran 1](./instalations_vm/06_install_ubuntu_screen.png)
![Installation Ubuntu – étape 2](./instalations_vm/06_install_ubuntu_2.png)
![Installation Ubuntu – étape 3](./instalations_vm/06_install_ubuntu_3.png)
![Installation Ubuntu – étape 4](./instalations_vm/06_install_ubuntu_4.png)
![Installation Ubuntu – étape 5](./instalations_vm/06_install_ubuntu_5.png)
![Installation Ubuntu – étape 6](./instalations_vm/06_install_ubuntu_6.png)
![Installation Ubuntu – étape 7](./instalations_vm/06_install_ubuntu_7.png)
![Installation Ubuntu – étape 8](./instalations_vm/06_install_ubuntu_8.png)
![Installation Ubuntu – étape 9](./instalations_vm/06_install_ubuntu_9.png)
![Installation Ubuntu – étape 10](./instalations_vm/06_install_ubuntu_10.png)
![Installation Ubuntu – étape 11](./instalations_vm/06_install_ubuntu_11.png)
![Installation Ubuntu – étape 12](./instalations_vm/06_install_ubuntu_12.png)
![Installation Ubuntu – étape 13](./instalations_vm/06_install_ubuntu_13.png)

---
## 1.3 Configuration de l’accès SSH à la machine virtuelle

Avant de commencer l’installation des dépendances (Docker, Kind, Kubectl, Helm), j’ai configuré un accès SSH afin de pouvoir travailler plus confortablement depuis mon ordinateur hôte.  
L’accès SSH permet notamment :

- d’utiliser un terminal plus lisible que celui de VMware ;
- de copier/coller facilement les commandes du TP ;
- d’ouvrir plusieurs sessions en parallèle si nécessaire ;
- de travailler comme sur un serveur distant réel.

---

### 1.3.1 Activation du service SSH

Lors de la première vérification, le service SSH était installé mais désactivé.  
Je l’ai activé puis démarré avec les commandes suivantes :

<!-- ligne code  -->
```bash 
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh

```


Après activation, le service passe correctement à l’état **active (running)** :

![Service SSH actif](./redirection_ssh/01_status_ssh_runing.png)

---

### 1.3.2 Récupération de l’adresse IP de la VM (mode NAT)

J’utilise le mode NAT de VMware.  
Pour récupérer l’adresse IP interne attribuée par VMware :

<!-- code  -->
``` bash
ip a
```


L’adresse IP apparaît sous l’interface `ens33`.

![Adresse IP de la VM](./redirection_ssh/02_vm_ip_address.png)

---

### 1.3.3 Configuration de la redirection NAT (port forwarding)

VMware utilise l’interface **VMnet8** pour le mode NAT.  
Pour exposer la VM via `127.0.0.1`, j’ai configuré une redirection du port **2222 → 22** :

- **Host port : 2222**
- **Virtual machine IP : 192.168.5.136** (IP obtenue précédemment)
- **Virtual machine port : 22 (SSH)**

#### Accès aux paramètres réseau

![Ouverture du Virtual Network Editor](./redirection_ssh/03_vmware_nat.png)

#### Sélection de VMnet8 (NAT)

![Sélection VMnet8](./redirection_ssh/04_vmware_nat.png)

#### Accès aux paramètres NAT

![Bouton NAT Settings](./redirection_ssh/05_vmware_nat.png)

#### Ajout de la règle de redirection

![Ajout du port 2222](./redirection_ssh/06_vmware_nat.png)


---

### 1.3.4 Connexion SSH via Git Bash (localhost)

Windows ne disposant pas du client OpenSSH, j’ai utilisé **Git Bash**, qui inclut un client SSH complet.  
La connexion s’effectue directement via localhost :

```` bash
ssh tyna@127.0.0.1 -p 2222
````

Après validation de la clé et saisie du mot de passe, la connexion réussit :

![Connexion SSH réussie](./redirection_ssh/07_vmware_nat.png)

---
# 2. Installation des dépendances

Avant de pouvoir créer un cluster Kubernetes et déployer Free5GC, plusieurs outils et composants doivent être installés sur la machine virtuelle Ubuntu.

Les dépendances nécessaires dans le cadre de ce TP sont :

- **Docker** : moteur de conteneurs utilisé par Kind pour créer les nœuds Kubernetes.
- **Kind (Kubernetes in Docker)** : permet de déployer un cluster Kubernetes léger basé sur des conteneurs.
- **Kubectl** : outil en ligne de commande permettant d’interagir avec le cluster Kubernetes.
- **Helm** : gestionnaire de charts utilisé pour déployer Free5GC et UERANSIM.
- **GTP5G Kernel Module** : module noyau indispensable pour activer le tunneling GTP-U et permettre le fonctionnement de l’UPF de Free5GC.
- **Dépôt `towards5gs-helm`** : charts Helm fournis dans le cadre du TP pour déployer automatiquement Free5GC et UERANSIM.

Les sous-sections suivantes détaillent la procédure complète d’installation de chacune de ces dépendances, avec les commandes exécutées et les captures d’écran associées.


---

## 2.2.1 Mise à jour du système

La première étape consiste à mettre à jour Ubuntu afin de garantir la compatibilité des paquets :

``` bash
sudo apt udate
sudo apt upgrade -y 
```

![Mise à jour du système](./docker/01_system_update.png)

---

## 2.2 Installation de Docker

Pour ce TP, j’ai installé Docker à partir des dépôts officiels Ubuntu, via le paquet `docker.io`.  
Cette méthode est simple, rapide et entièrement compatible avec l’utilisation de Kind.

Installation :



``` bash
 sudo apt install-y docker.io
```
![Installer docker](./docker/02_install_docker.png)

Vérification de la version installée :

``` bash
docker --version
```
![Installer docker](./docker/02_02_installdocker.png)


Activation du Service Docker
``` bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

![Docker installé et actif](./docker/03_docker_status.png)

## 2.3 Installation de Kind

Comme indiqué dans le TP, Kind doit être installé manuellement via un binaire placé dans le répertoire `/usr/local/bin`.  
Kind (*Kubernetes in Docker*) permet de créer un cluster Kubernetes directement à l’intérieur de Docker.

### Téléchargement de Kind

```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
```
Rendre le binaire exécutable

```bash
chmod +x ./kind
```
Déplacement dans le dossier /usr/local/bin
```bash
sudo mv ./kind /usr/local/bin/kind
```

Vérification de l’installation

```bash
kind --version
```
![Installer kind](./dependances/01_installation_Kind.png)
## 2.4 Installation de Kubectl

Télécharger Kubectl
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
Rendre exécutable
```bash
chmod +x kubectl
```
Déplacer dans /usr/local/bin

```bash
sudo mv kubectl /usr/local/bin/
```
Vérifier l'installation
```bash
kubectl version --client

```
![Installer kubectl](./dependances/02_installation_kubectl.png)



## 2.5 Instalaltion de Helm

Installation Helm via script officiel 
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```
Vérifier l'installation
```bash
helm version
```
![Installer helm](./dependances/03_installation_helm.png)
## 2.6 Installation du module noyau GTP5G
L’installation du module noyau **gtp5g** est indispensable pour permettre le routage GTP-U utilisé par l’UPF dans Free5GC.  
Ce module permet la gestion des tunnels et du forwarding des paquets sur les interfaces N3 et N6.

Installer les outils de compilations
```bash
sudo apt install -y git make gcc linux-headers-$(uname -r)
```
Télécharger GTP5G
```bash
git clone https://github.com/free5gc/gtp5g.git
```
Compiler le module noyau
```bash
cd gtp5g
make clean
make
```
Installer le module 
```bash
sudo make install 
```
Vérifier que le module est bien installé 
```bash
lsmod | grep gtp5g
```
![Instllation du noyau GTP5](./dependances/04_installation_noyau5G.png)


## 2.7 Clonage du dépôt towards5gs-helm
Le TP utilise le dépôt GitHub *towards5gs-helm*, qui contient l’ensemble des charts Helm nécessaires pour déployer Free5GC et UERANSIM dans le cluster Kubernetes.

Parmis les étapes à suivre :

Se mettre dans le répertoir $HOME

```bash
cd ~
```
Cloner le dépôt depuis GitHub 
```bash
git clone https://github.com/cdestre/towards5gs-helm.git
# Vérifier qu'il existe
ls
```
![clonage du depot](./dependances/05_git_clone_repo.png)

# 3. Création du cluster KinD

Après l’installation des dépendances, l’étape suivante consiste à créer un cluster Kubernetes basé sur Kind.  
Le TP impose l’utilisation d’un fichier de configuration pour définir la structure du cluster, composé de deux nœuds :

- un nœud **control-plane**
- un nœud **worker**

---

## 3.1 Création du fichier de configuration `mycluster.yaml`

```bash
nano mycluster.yaml
```
Le fichier de configuration suivant décrit la topologie du cluster :

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker


```
![création fichier yaml](./cluster/01_mycluster_yaml.png)

Création du cluster kind

```bash
sudo kind create cluster --config mycluster.yaml
```
![création cluster kind](./cluster/02_creation_cluster.png)

Vérification des nœuds Kubernetes:

```bash
# toujours mettre sudo
sudo kubectl get nodes 
```
![nodes](./cluster/03_nodes_kubernetes.png)
## 3.4 Installation manuelle des plugins CNI
Comme indiqué dans le TP, Kind ne fournit pas de plugins CNI par défaut.  
Il est donc nécessaire de télécharger et d’installer manuellement les plugins réseaux utilisés par Kubernetes.

### Téléchargement et extraction des plugins CNI

```bash
wget https://github.com/containernetworking/plugins/releases/download/v1.6.0/cni-plugins-linux-amd64-v1.6.0.tgz

# Puis extraire 
tar -xvf cni-plugins-linux-amd64-v1.6.0.tgz

```
![telechargerment_cni](./cluster/04_cni_extract.png)

### Identification des conteneurs Kind


```bash
sudo docker ps
```
![les_ids](./cluster/05_docker_ps_nodes.png)
Une fois les IDs récupérés, j’ai copié les plugins CNI vers chacun des nœuds :
```bash
sudo docker cp . 97199854493c:/opt/cni/bin/  #control
sudo docker cp . efced09c7e28:/opt/cni/bin/ #worker

```
![docker_cp_cni](./cluster/06_docker_cp_cni.png)

Vérification des interfaces réseau du worker 

```bash
sudo docker exec -it efced09c7e28 /bin/bash


# À l'intérieur du conteneur
ip a
ip r

exit

```
Cette étape est importante car les informations récupérées seront utilisées plus tard lors de la configuration de Free5GC — notamment pour l’UPF, qui doit connaître :

- la gateway du worker : **172.18.0.2**
- l’interface du worker pour le réseau N6 : **eth0**

Et l’interface eth0 du worker pour la N6
![addr_ip_worker](./cluster/07_worker_ip.png)
---


## 4. Installation de Multus CNI

Free5GC nécessite plusieurs interfaces réseau pour pouvoir communiquer sur les différents plans (N2, N3, N4, N6).  
Pour cela, le TP impose l’installation du plugin **Multus CNI**, qui permet d’attacher plusieurs interfaces réseau à un même pod.

---

## 4.1 Déploiement de Multus

Comme demandé dans le TP, j’ai commencé par cloner le dépôt officiel de Multus :

```bash
git clone https://github.com/k8snetworkplumbingwg/multus-cni
cd multus-cni
cat ./deployments/multus-daemonset-thick.yml | sudo kubectl apply -f -
```
Cette commande déploie :

 - le CRD network-attachment-definitions

 - les rôles RBAC nécessaires

 - le ConfigMap de Multus

 - le DaemonSet kube-multus-ds
![installation_multus](./cluster/08_multus_node_annotation.png)
## 4.2 Vérification du DaemonSet Multus

Pour vérifier que Multus est bien installé sur les nœuds :

```bash
sudo kubectl describe node kind-worker | grep -i multus
```
- La présence du pod kube-multus-ds-xxxx confirme que Multus fonctionne.
![multus_ready](./cluster/multus_fonctionne.png)
## 4.3 Vérification des fichiers CNI sur les nœuds

Après le déploiement de Multus, il est important de vérifier que les fichiers de configuration CNI sont bien présents sur chacun des nœuds du cluster Kind.  
Ces fichiers confirment que :

- **Kindnet** est bien installé comme CNI primaire.
- **Multus** est bien installé comme méta-CNI permettant les interfaces secondaires.

Les commandes suivantes permettent d’inspecter le contenu du répertoire `/etc/cni/net.d/` de chaque nœud :

```bash
# Nœud control-plane
sudo docker exec -it 97199854493c ls /etc/cni/net.d/

# Nœud worker
sudo docker exec -it efced09c7e28 ls /etc/cni/net.d/
```
![file_config](./cluster/09_deploy_multus.png)

# 5. Installation et configuration de Free5GC

Dans cette section, nous installons Free5GC sur un cluster Kubernetes KinD, après avoir préparé :

- le stockage MongoDB

- le réseau Multus N6

- les valeurs UPF spécifiques à KinD

- le namespace free5gc

* L'installation se fait entièrement via Helm.

## 5.1 Installation de Multus CNI
Multus permet l’attachement de plusieurs interfaces réseau à un pod, ce qui est indispensable pour l’UPF de Free5GC.
, c'est une partie déjà fini sur la section de haut 

## 5.2 Préparation du stockage MongoDB
Free5GC utilise MongoDB comme base de données.
Pour garantir un déploiement stable et fonctionnel, il est nécessaire de préparer correctement le chart MongoDB avant l’installation de Free5GC.

Cette section présente les bonnes pratiques, sous forme de sous-étapes claires :

- Vérifier et extraire le sous-chart MongoDB

- Modifier les paramètres de l’image MongoDB

- Créer le dossier de stockage local dans le worker

- Créer et appliquer un PersistentVolume

- Vérifier que le PV est bien pris en compte

### 5.2.1 Vérification et extraction du sous-chart MongoDB

- Le chart Free5GC contient un sous-chart mongodb situé dans :
```bash
cd ~/towards5gs-helm/charts/free5gc/charts

# Dans certains cas, ce sous-chart est fourni sous forme d’archive .tgz.
# Il faut alors le décompresser :

cd ~/towards5gs-helm/charts/free5gc/charts
tar -xvf mongodb-*.tgz

```
![deziper_mongo](./5Gfree/05_dezip_mongo.png)

Après extraction, on vérifie la présence du dossier mongodb/, contenant notamment :
- values.yaml (fichier de configuration principal)
- templates/ (définition du StatefulSet MongoDB)

![info_mongodb](./5Gfree/05_.2.png)
#### 5.2.1.1 Modification du fichier values.yaml  
Ouvrir le fichier :
```bash
nano mongodb/values.yaml

```
Au début du fichier, la configuration par défaut utilise :

```yaml
 ## Au début 
image:
  registry: docker.io
  repository: bitnami/mongodb
  tag: 4.4.4-debian-10 # ici faut changer l'image 
```
 ![structure_mongo](./5Gfree/01_etat_ini_mongo.png)
- Par :
```yaml
repository: mongo
tag: 4.4
```
![solution_monfdb](./5Gfree/solution_mongo.png)

<!-- ###################################### -->
### 5.2.2 Création du dossier de stockage dans le worker
MongoDB nécessite un volume persistant.
Dans un cluster KinD, ce stockage doit être créé manuellement dans le conteneur du nœud worker.
```bash
sudo docker exec -it kind-worker mkdir /home/kubedata

```
### 5.2.3 Création et application du PersistentVolume (PV)
- Dans $HOME  créer le fichier suivant : 
```bash
~> nano volume.yaml

```
- Avec le contenu suivant :
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-local-pv9
  labels:
    project: free5gc
spec:
  capacity:
    storage: 8Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  local:
    path: /home/kubedata
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - kind-control-plane

```
- Appliquer le volume 
```bash
sudo kubectl apply -f volume.yaml
sudo kubectl get pv
```
![pv_yes](./5Gfree/01_pv.png)

## 5.3 Définition du réseau N6 (Multus)
Le réseau N6 correspond à l’interface de sortie de l’UPF vers le Data Network (Internet ou LAN).
Multus est utilisé pour attacher une interface réseau supplémentaire au pod UPF.
- Créer le fichier :
```bash
# créer le fichier n6network.yaml dans le home
nano n6network.yaml
```
- Avec le contenu :
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: n6network
  namespace: free5gc
spec:
  config: '{
    "cniVersion": "0.3.1",
    "type": "ipvlan",
    "mode": "l2",
    "master": "eth0",
    "ipam": {
      "type": "static",
      "addresses": [
        {
          "address": "172.18.0.22/16",
          "gateway": "172.18.0.1"
        }
      ]
    }
  }'

```
- Appliquer 
```bash
# création du namespace free5gc
sudo kubectl create ns free5gc

# appliquer le réseau N6
sudo kubectl apply -f n6network.yaml --validate=false

# vérifier avec cette commande  seulement les NAD de ce namespace (free5gc)
sudo kubectl get network-attachment-definitions -n free5gc


```
![n6](./5Gfree/02_N6.png)






## 5.4 Configuration spécifique Free5GC (UPF + réseaux)
### 5.4.1 Identification du réseau KinD
Pour configurer correctement l’UPF, nous devons connaître :

- l’interface réseau du worker

- l’adresse IP du worker

- la gateway Docker/Kind

Commande :
```bash
sudo docker exec -it kind-worker bash
ip a
ip r
exit
```
Résultat observé comme vu precédement :

- interface principale : eth0

- réseau Kind : 172.18.0.0/16

- gateway : 172.18.0.1
### 5.4.2 Configuration de l’UPF (free5gc-upf/values.yaml)
Dans le chart Free5GC, chaque fonction réseau possède son propre sous-chart.
L’UPF doit obligatoirement connaître l’IP de son interface N6 :
```bash
cd ~/towards5gs-helm/charts/free5gc/charts/free5gc-upf/
nano values.yaml
```
- Modifier :
```yaml
n6if:
  ipAddress: 172.18.0.22
```
![n6if](./5Gfree/n6if.png)
### 5.4.3 Configuration du N6 dans le chart Free5GC
En plus de la création du fichier n6network.yaml, il est nécessaire d’informer Free5GC qu’il doit utiliser ce réseau Multus pour l’interface N6 de l’UPF.

Pour cela, il faut modifier le fichier values.yaml du chart principal Free5GC.
- Étape 1 : ouvrir le fichier
```bash
cd ~/towards5gs-helm/charts/free5gc
nano values.yaml

```
- Étape 2 : localiser la section n6network:
```yaml
n6network:
  enabled: true
  name: n6network

```
- Étape 3 : Les champs suivants doivent être modifiés en fonction du réseau observé sur le worker :
```yaml
n6network:
  enabled: true
  name: n6network               # doit être identique au NAD créé ("n6network")
  type: ipvlan
  masterIf: eth0                # interface observée dans le worker
  subnetIP: 172.18.0.0          # sous-réseau déduit du worker
  cidr: 16                      # masque (/16 → 255.255.0.0)
  gatewayIP: 172.18.0.1         # gateway observée dans 'ip r'
  exludeIP: 172.18.0.0
```
![n6netwpojr](./5Gfree/n6network.png)

## 5.5 Installation de Free5GC via Helm
Une fois le stockage MongoDB et le réseau Multus configurés, nous pouvons installer Free5GC grâce au chart Helm fourni dans le dépôt towards5gs-helm.

- Le chart Free5GC installe automatiquement les fonctions suivantes :
  - AMF -> Network 2
  - SMF -> Network 4 
  - UPF (avec l’interface N6) -> Network 6
  - AUSF
  - UDM
  - UDR
  - PCF
  - NRF
  - WebUI

- Commande d’installation :
```bash
# Il  faut obligatoirement se situer sur ce repertoir 
cd ~/towards5gs-helm/charts

# Installation dans le namespace free5gc
sudo helm -n free5gc install free5gc-premier ./free5gc/


# Ensuite, vérifier l’état des pods :
sudo kubectl get pods -n free5gc

```
![deziper](./5Gfree/05_dezip_mongo.png)
ensuite chercher le dossier mongodb afin de modifier le fichier value 
![mongo](./5Gfree/05_.2.png)
 modifier le fichier volume.yaml
Avant : 
![volume](./5Gfree/06_mongo_2.png)
Aprés modification 
![volume2](./5Gfree/06_value_mongo.png)
ensuite on recpee l'image récente mongo:4.4
```bash
# Télécharger l'image dans la vm 
sudo docker pull mongo:4.4
# Charger l'image dans le cluster kind
sudo kind load docker-image mongo:4.4 --name kind

# Supprimer l’ancien StatefulSet MongoDB
sudo kubectl delete pvc -n free5gc --all
sudo kubectl delete pv example-local-pv9

# Partie sur $HOME
cd 

sudo kubectl apply -f volume.yaml

# ensuite reinstaller free5GC
cd ~/towards5gs-helm/charts
sudo helm -n free5gc install free5gc-premier ./free5gc/

```
## 5.5 Installation de Free5GC via Helm
- Une fois MongoDB prêt, Multus configuré et l’UPF ajusté, on peut lancer l’installation :
```bash
# Toujours se mettre sur ce repertoir

cd ~/towards5gs-helm/charts

sudo helm -n free5gc install free5gc-premier ./free5gc/

# Vérification :
sudo kubectl get pods -n free5gc
```
![mongo_run](./5Gfree/07_mongo_run.png)


## 5.7 – Accès à l’interface WebUI de Free5GC
Lorsque les pods Free5GC sont en état Running , nous pouvons accéder à l’interface WebUI afin de visualiser les UEs, les sessions PDU et la configuration du Core.

### 5.7.1 Préparation : créer un tunnel SSH SOCKS5
Pour accéder au WebUI depuis notre machine Windows, nous utilisons un tunnel SSH via l’option -D (SOCKS proxy).

- Dans git bash 
```bash
ssh -D 8080 -N tyna@127.0.0.1 -p 2222
```
![tunnel_ssh](./5Gfree/vm_pour%20lancer%20navigateur.png)
Cette commande :

- crée un proxy SOCKS5 local sur 127.0.0.1:8080

- sans ouvrir de session (-N)

- en utilisant le port SSH redirigé via VMware (-p 2222)
### 5.7.2 Configuration du navigateur
Nous devons indiquer à Firefox/Chrome dans pramètre réseau  d’utiliser le proxy SOCKS5 :

- Adresse : 127.0.0.1

- Port : 8080

- Type : SOCKS5

- Activer Proxy DNS
![proxy](./5Gfree/proxy.png)
### 5.7.3 Accès au WebUI
- Une fois le proxy configuré, nous pouvons accéder à l’interface :
```cpp
http://172.18.0.2:5000

```
- Cette adresse correspond à l’IP du nœud worker du cluster KinD obtenue avec :
```bash
sudo docker exec -it kind-worker ip a
```
### 5.7.4 Connexion au WebUI
Identifiants par défaut du chart :

- username : admin

- password : free5gc
![web_ui](./5Gfree/ue_final.png)
### 5.7.5 Ajout d’un utilisateur UE (WebUI)
L'étape suivante consiste à créer un utilisateur (IMSI, clé, OPc, etc.).
Dans WebUI → Subscriber Management, j’ai ajouté le profil d’UE fourni dans l’énoncé.
![subscribe](./5Gfree/final_subscribe_1ure%20d'éc.png)
![subscribe](./5Gfree/final_subscribe_2.png)


Dans l’interface Subscriber Management, on voit que l’utilisateur a bien été ajouté dans Free5GC (profil IMSI, clés, OPc, etc.).
Cependant, l’UE n’est pas encore enregistré dans le réseau 5G :
l’inscription (registration) ne sera effectuée que lorsque UERANSIM simulera effectivement l’UE.
À ce stade, l’utilisateur est présent dans la base de données, mais pas encore attaché au cœur 5G.
## 5.8 – Déploiement d’UERANSIM (gNB + UE simulés)
Pour permettre l’enregistrement de l’UE dans Free5GC, nous devons déployer le simulateur RAN UERANSIM (gNB + UE).

Il est important d’installer UERANSIM dans le même namespace que Free5GC, c’est-à-dire free5gc, afin que les composants puissent communiquer correctement.

### 5.8.1 Installation du chart UERANSIM

- Comme pour Free5GC, l’installation se fait depuis le répertoire charts :
```bash
cd ~/towards5gs-helm/charts
sudo helm -n free5gc install ueransim-premier ./ueransim/
```
Cette commande installe automatiquement :

- un gNB simulé,

- un UE simulé,

- configure les communications NGAP/NRR entre gNB ↔ AMF.
![create_ue](./5Gfree/ueransim_ccreate.png)

### 5.8.2 Vérification des pods UERANSIM
- Une fois l’installation terminée :
```bash
sudo kubectl get pods -n free5gc

```
![ue_run](./5Gfree/5_7resultat_ue.png)

![registrer$d](./5Gfree/registred_final.png)

![registrer$d](./5Gfree/registred_part1.png)

![registrer$d](./5Gfree/registred_part2.png)

### 5.8.3 Lien avec Free5GC
Une fois UERANSIM lancé, plusieurs échanges automatiques s’effectuent entre le gNB, l’UE et le cœur Free5GC :

- le gNB envoie un NG Setup Request à l’AMF pour établir le lien RAN ↔ Core ;

- l’UE démarre une procédure 5G Registration auprès de l’AMF ;

- Free5GC effectue l’Authentication et le Security Mode Control ;

- l’UE reçoit un Registration Accept et devient enregistré ;

- lorsqu’une PDU Session est créée, une interface virtuelle uesimtun0 apparaît dans le pod UE, représentant le tunnel GTP-U vers l’UPF.

Ces échanges seront vérifiés dans la section suivante à l’aide des tests fonctionnels.
# 6 Tests de validation du réseau 5G Free5GC
 Après le déploiement complet du cœur 5G (AMF, SMF, UPF, NRF, UDM, UDR, PCF) et du RAN simulé UERANSIM (gNB + UE), nous validons maintenant le fonctionnement du réseau :

Plan de contrôle : Registration 5G, Authentication, Security Mode Control

Plan utilisateur : établissement d’une PDU session, création de uesimtun0, trafic vers l’UPF

Les tests suivants prouvent que l’UE est correctement enregistré et que le tunnel GTP-U fonctionne.


## 6.1  Récupération du nom du pod UE
 ```bash
 export POD_NAME=$(sudo kubectl get pods --namespace free5gc -l "component=ue" -o jsonpath="{.items[0].metadata.name}")
 ```
- Explication :
  - Cette commande récupère automatiquement le nom du pod UERANSIM-UE grâce au label component=ue et le stocke dans la variable $POD_NAME.

Résultat obtenu dans notre environnement :
 ![var_pod](./5Gfree/var_pod_name.png)

## 6.2 Vérification de la création de l’interface TUN côté UE
### 6.2.1 Lecture des logs UE

```bash
sudo kubectl --namespace free5gc logs $POD_NAME

```
- Explication :
  - Ce que l’on vérifie dans les logs :

  - NG Setup entre gNB ↔ AMF

  - Registration Request → Accept

  - Authentication OK

  - Security Mode OK

  - Création d’une session PDU

 ![var_pod_in](./5Gfree/6.2.1_init.png)

### 6.2.2 Vérification de l’interface uesimtun0

```bash
sudoo kubectl --namespace free5gc exec -it $POD_NAME -- ip address
```

- Explication :
  - Cette commande permet de vérifier la présence et l’état de l’interface uesimtun0, qui symbolise le tunnel GTP/UPF → UE.

- Résultat obtenu :

 ![var_pod_in](./5Gfree/ue_ths.png)
 Conclusion : l’UE est attaché et dispose d’une IP 5G.

## 6.3 Test du trafic utilisateur (plan utilisateur - UP)

### 6.3.1 Ping vers l’interface N6 de l’UPF
```bash
sudo kubectl --namespace free5gc exec -it $POD_NAME -- ping -I uesimtun0 172.18.0.22
```
- Objectif : vérifier que les paquets sortent bien via le tunnel 5G et atteignent l’UPF sur son interface N6.

![test_cnx](./5Gfree/test-6.3.png)
Conclusion : la session PDU est active et fonctionnelle.

# 6.4 Test de sortie Internet
```bash
sudo kubectl --namespace free5gc exec -it $POD_NAME -- ping -I uesimtun0 www.google.com
```

![google](./5Gfree/test_google.png)
Oui, ton rapport est **logique**, **cohérent**, et ce que tu veux ajouter dans la partie “Erreurs rencontrées” est **parfaitement pertinent**.

Voici comment organiser proprement cette section *exactement comme dans un vrai rapport de TP Free5GC*, en intégrant :

* l’erreur sur MongoDB
* l’erreur sur le cluster KinD
* l’erreur sur `$POD_NAME`
* l’erreur `ImagePullBackOff`
* l’erreur UPF (normal avant configuration N6)
* l’erreur NAT/Google (optionnelle)

Je te donne une version **propre, structurée, prête à coller dans ton rapport**, en reprenant **tes captures**, **tes textes**, mais corrigé, organisé et cohérent.

---

# 7 Problèmes rencontrés et solutions apportées

Cette section présente les principales erreurs rencontrées lors du déploiement de Free5GC et d’UERANSIM, ainsi que les solutions mises en place pour aboutir à un fonctionnement complet du réseau 5G.

---

## 7.1 Problème : MongoDB en `ImagePullBackOff`
![err](./5Gfree/01_mongodb.png)

### **Symptôme**

Lors de l’installation initiale du chart Free5GC, MongoDB reste en :

```
ImagePullBackOff
```

### **Cause**

Le sous-chart MongoDB utilise par défaut l’image **bitnami/mongodb**, qui n’est plus compatible avec l’environnement KinD (permissions, user non-root, probes trop strictes).

### **Analyse du fichier `values.yaml`**

Configuration initiale :

```yaml
image:
  registry: docker.io
  repository: bitnami/mongodb
  tag: 4.4.4-debian-10
```

Et des probes trop strictes :

```yaml
livenessProbe:
  enabled: true
readinessProbe:
  enabled: true
```

### **Solution**

1. Dézipper le sous-chart MongoDB
2. Modifier `mongodb/values.yaml` :

```yaml
repository: mongo
tag: 4.4
livenessProbe:
  enabled: false
readinessProbe:
  enabled: false
```

3. Charger manuellement l’image dans KinD :

```bash
sudo docker pull mongo:4.4
sudo kind load docker-image mongo:4.4 --name kind
```

4. Recréer le PV + réinstaller Free5GC.

### **Résultat**

MongoDB passe en état :

```
Running
```

- Free5GC peut démarrer correctement.

---

## 7.2 Explication du test de connectivité UE → UPF
Voici **exactement ce que tu m’as demandé** :
➡️ **le texte en Markdown**, propre, clair, prêt à coller dans ton rapport
➡️ **basé uniquement sur TA capture**, avec une explication simple pour un débutant.

Tu n’as qu’à copier-coller :

---

# 📌 **Explication du test de connectivité UE → UPF**

### **Capture :**

![test\_ue](./5Gfree/7.ping.jpg)

---

 **Analyse et explication**

Cette capture montre les trois étapes essentielles du fonctionnement du tunnel 5G entre l’UE (UERANSIM) et le UPF de Free5GC.

---

 **1️ Premier test : Ping trop tôt → erreur normale**

Commande exécutée :

```bash
sudo kubectl --namespace free5gc exec -it $POD_NAME -- ping -I uesimtun0 www.google.com
```

- Résultat :

```
ping: usage error: Destination address required
command terminated with exit code 1
```

###  Pourquoi cette erreur apparaît ?

À ce moment :

* l’UE **n’était pas encore enregistré** dans Free5GC (Registration incomplet),
* **aucune Session PDU** n’était encore établie,
* l’interface virtuelle **uesimtun0 n’existait pas encore**.

**C’est normal : aucun ping n’est possible tant que l’UE n’a pas obtenu son IP 5G.**

---

 **2️ Deuxième test : Vérification des interfaces réseau**

Commande :

```bash
kubectl exec -it -n free5gc $POD_NAME -- ip addr
```

- Résultat :
On voit apparaître l’interface :

```
3: uesimtun0: ...
    inet 10.1.0.1/32 scope global uesimtun0
```

###  Ce que cela signifie

* L’UE a maintenant **reçu une adresse IP 5G** (`10.1.0.1`),
* Le tunnel GTP-U entre le gNB → UPF → UE est **créé**,
* La session PDU est **active**, donc le plan utilisateur fonctionne.

- **L’UE est correctement attaché au réseau 5G.**

---

 **3️ Troisième test : Ping du UPF (interface N6) → Succès**

Commande :

```bash
sudo kubectl exec -it -n free5gc $POD_NAME -- ping -I uesimtun0 172.18.0.22
```

- Résultat :

```
64 bytes from 172.18.0.22: time=1.32 ms
64 bytes from 172.18.0.22: time=1.94 ms
64 bytes from 172.18.0.22: time=5.27 ms
```

###  Interprétation

* `172.18.0.22` = adresse N6 du UPF
* Le ping passe **via uesimtun0**
* Les temps de réponse sont faibles et stables

 **Preuve que le trafic passe correctement par le UPF.**


![err_net](./5Gfree/7.ping.jpg)

### **Symptôme**

La variable montre un nom incorrect :

```
ueransim-premier-ue-6fdbccb8cf-pjtqr
```

Au lieu de celui visible avec :

```bash
kubectl get pods -n free5gc
```

### **Cause**

Le label utilisé dans la commande était erroné.

### **Solution**

Utiliser le bon label du chart UERANSIM :

```bash
export POD_NAME=$(kubectl get pods -n free5gc -l "component=ue" -o jsonpath="{.items[0].metadata.name}")
```

### **Résultat**

La variable correspond bien au pod UE actif.

---

## 7.3 Problème : Erreur à la création du cluster KinD

### **Symptôme**

Après création :

```
kubectl get nodes
→ connection refused 127.0.0.1:XXXXX
```

### **Cause**

Le fichier kubeconfig n’était pas appliqué (bug fréquent de KinD).

### **Solution**

Redémarrer l’API ou recréer le cluster :

```bash
kind delete cluster
kind create cluster --config mycluster.yaml
```

---

## 7.4 Problème : UPF en `CrashLoopBackOff`

![loop-upf)](./5Gfree/Final_mongo_db_run.png)

### **Symptôme**

```
free5gc-upf → CrashLoopBackOff
```

### **Cause**

Normal :
 l’UPF ne peut pas se lancer tant que **N6 n’est pas configuré** :

* `NetworkAttachmentDefinition` manquante
* `n6if.ipAddress` non renseigné

### **Solution**

1. Créer le fichier **n6network.yaml**
2. Modifier dans `free5gc-upf/values.yaml` :

```yaml
n6if:
  ipAddress: 172.18.0.22
```

3. Activer N6 dans `free5gc/values.yaml`

```yaml
n6network:
  enabled: true
  name: n6network
  subnetIP: 172.18.0.0
  cidr: 16
  gatewayIP: 172.18.0.1
```

### **Résultat**

L’UPF passe en **Running**, et UERANSIM peut s’enregistrer.

---
