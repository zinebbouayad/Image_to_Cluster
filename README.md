# 🚀 Atelier — From Image to Cluster

**Auteur :** zinebbouayad  
**Environnement :** GitHub Codespaces  
**Outils :** Packer · Kubernetes K3d · Ansible · Docker · Nginx

---

## 📋 Description

Cet atelier industrialise le cycle de vie d'une application web simple en :

1. Construisant une image Docker personnalisée **Nginx** avec **Packer**
2. Déployant cette image sur un cluster **Kubernetes K3d** via **Ansible**
3. Le tout dans un environnement reproductible via **GitHub Codespaces**

---

## 🏗️ Architecture cible

```
GitHub Codespace
│
├── index.html            ← Page web personnalisée (artefact applicatif)
│
├── Packer                ← Build de l'image Docker
│   └── nginx.pkr.hcl     → image mon-nginx-custom:latest
│           ↓
├── K3d                   ← Import de l'image dans le cluster local
│   └── k3d image import
│           ↓
└── Kubernetes            ← Déploiement de l'application
    ├── deployment.yaml   → 1 Pod nginx-custom en cours d'exécution
    └── service.yaml      → Service NodePort exposé
            ↓
      port-forward 8081 → navigateur ✅
```

---

## 📦 Prérequis

- Un compte **GitHub**
- Un **Codespace GitHub** ouvert sur ce repository
- Aucune installation locale nécessaire — tout se passe dans le Codespace

---

## 🔧 Séquence 1 — Création du Codespace

1. Forkez ce repository sur votre compte GitHub
2. Depuis l'onglet **[CODE]**, cliquez sur **"Open with Codespaces"**
3. Attendez que l'environnement soit prêt (~1 minute)

---

## ☸️ Séquence 2 — Création du cluster Kubernetes K3d

### Installation de K3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

### Création du cluster (1 master + 2 workers)

```bash
k3d cluster create lab \
  --servers 1 \
  --agents 2
```

### Vérification du cluster

```bash
kubectl get nodes
```

Résultat attendu :

```
NAME               STATUS   ROLES                  AGE   VERSION
k3d-lab-agent-0    Ready    <none>                 ...   v1.31.5+k3s1
k3d-lab-agent-1    Ready    <none>                 ...   v1.31.5+k3s1
k3d-lab-server-0   Ready    control-plane,master   ...   v1.31.5+k3s1
```

### Test avec l'application Mario (optionnel)

```bash
kubectl create deployment mario --image=sevenajay/mario
kubectl expose deployment mario --type=NodePort --port=80
kubectl port-forward svc/mario 8080:80 &
```

Dans l'onglet **[PORTS]** du Codespace, rendez le port `8080` **public** et ouvrez l'URL.

---

## 🔨 Séquence 3 — Build Packer + Déploiement K3d via Ansible

### Vue d'ensemble du processus

```
[index.html] → Packer build → [mon-nginx-custom:latest] → k3d import → Ansible deploy → [Pod Running] → port-forward → [Navigateur]
```

---

### Étape 1 — Vérifier les outils installés

Packer et Ansible sont déjà disponibles dans le Codespace :

```bash
packer version
ansible --version
```

---

### Étape 2 — Créer le template Packer

Créez le fichier `nginx.pkr.hcl` à la racine du repository :

```hcl
packer {
  required_plugins {
    docker = {
      source  = "github.com/hashicorp/docker"
      version = "~> 1"
    }
  }
}

source "docker" "nginx" {
  image  = "nginx:latest"
  commit = true
}

build {
  sources = ["source.docker.nginx"]

  provisioner "file" {
    source      = "index.html"
    destination = "/usr/share/nginx/html/index.html"
  }

  post-processors {
    post-processor "docker-tag" {
      repository = "mon-nginx-custom"
      tags       = ["latest"]
    }
  }
}
```

> **Explication :**
> - `commit = true` : Packer crée l'image directement dans Docker (format standard)
> - `provisioner "file"` : copie votre `index.html` dans le conteneur
> - `docker-tag` : nomme l'image `mon-nginx-custom:latest`

---

### Étape 3 — Builder l'image avec Packer

```bash
packer init nginx.pkr.hcl
packer build nginx.pkr.hcl
```

Vérifiez que l'image est bien créée :

```bash
docker images | grep mon-nginx
```

Résultat attendu :

```
mon-nginx-custom   latest   abc123...   2 minutes ago   186MB
```

---

### Étape 4 — Importer l'image dans K3d

> ⚠️ **Cette étape est indispensable** : K3d est un cluster isolé qui ne voit pas les images Docker locales par défaut. Il faut les importer explicitement.

```bash
k3d image import mon-nginx-custom:latest -c lab
```

Résultat attendu :

```
INFO[0000] Importing image(s) into cluster 'lab'
INFO[0020] Successfully imported 1 image(s) into 1 cluster(s)
```

---

### Étape 5 — Créer le fichier de déploiement Kubernetes

Créez le fichier `deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-custom
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-custom
  template:
    metadata:
      labels:
        app: nginx-custom
    spec:
      containers:
      - name: nginx-custom
        image: mon-nginx-custom:latest
        imagePullPolicy: Never
        ports:
        - containerPort: 80
```

> ⚠️ **Important** : `imagePullPolicy: Never` est obligatoire pour que Kubernetes utilise l'image locale importée, et non tenter de la télécharger depuis Docker Hub.

---

### Étape 6 — Déployer sur K3d via Ansible

Créez le fichier `deploy.yml` (playbook Ansible) :

```yaml
---
- name: Deploy nginx-custom on K3d
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:

    - name: Appliquer le Deployment
      shell: kubectl apply -f deployment.yaml

    - name: Attendre que les pods soient prêts
      command: kubectl rollout status deployment/nginx-custom --timeout=90s

    - name: Afficher les pods
      command: kubectl get pods -l app=nginx-custom
      register: pods
    - debug: var=pods.stdout_lines

    - name: Créer le Service NodePort
      shell: kubectl expose deployment nginx-custom --type=NodePort --port=80 --dry-run=client -o yaml | kubectl apply -f -
```

Lancez le déploiement :

```bash
ansible-playbook deploy.yml -i inventory.ini \
  -e "ansible_python_interpreter=/usr/bin/python3"
```

Résultat attendu :

```
TASK [Attendre que les pods soient prêts]
deployment "nginx-custom" successfully rolled out

TASK [Afficher les pods]
NAME                            READY   STATUS    RESTARTS   AGE
nginx-custom-xxx                1/1     Running   0          5s
```

---

### Étape 7 — Exposer l'application et y accéder

```bash
kubectl port-forward svc/nginx-custom 8081:80 &
```

Testez dans le terminal :

```bash
curl http://localhost:8081
```

Puis dans l'onglet **[PORTS]** du Codespace :
- Localisez le port **8081**
- Clic droit → **Port Visibility** → **Public**
- Cliquez sur l'icône 🌐 pour ouvrir l'URL dans le navigateur

✅ Vous devriez voir votre page `index.html` personnalisée s'afficher !

---

## ✅ Résultat final

![Nginx déployé via Packer + Ansible sur K3d](Architecture_cible.png)

```
✅ Nginx déployé via Packer + Ansible sur K3d
Build time: 2026-04-24T10:40:58+00:00
```

---

## 📁 Structure du projet

```
Image_to_Cluster/
├── index.html          # Page web personnalisée servie par Nginx
├── nginx.pkr.hcl       # Template Packer pour builder l'image Docker
├── deployment.yaml     # Manifeste Kubernetes pour déployer sur K3d
├── deploy.yml          # Playbook Ansible pour automatiser le déploiement
├── inventory.ini       # Inventaire Ansible (localhost)
├── service.yaml        # Service Kubernetes NodePort
└── README.md           # Ce fichier
```

---

## 🛠️ Récapitulatif des commandes

| Étape | Commande |
|-------|----------|
| Installer K3d | `curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh \| bash` |
| Créer le cluster | `k3d cluster create lab --servers 1 --agents 2` |
| Vérifier le cluster | `kubectl get nodes` |
| Build image Packer | `packer build nginx.pkr.hcl` |
| Vérifier l'image | `docker images \| grep mon-nginx` |
| Importer dans K3d | `k3d image import mon-nginx-custom:latest -c lab` |
| Déployer via Ansible | `ansible-playbook deploy.yml -i inventory.ini` |
| Port-forward | `kubectl port-forward svc/nginx-custom 8081:80 &` |
| Tester | `curl http://localhost:8081` |

---

## ⚠️ Problèmes rencontrés et solutions

### Problème 1 : `export_path` incompatible avec `docker-tag`
**Erreur :** `Post-processor failed: Unknown artifact type: packer.docker`  
**Solution :** Remplacer `export_path` par `commit = true` dans le template Packer.

### Problème 2 : `ErrImagePull` / `ImagePullBackOff`
**Erreur :** Kubernetes essaie de télécharger l'image depuis Internet.  
**Solution :** Ajouter `imagePullPolicy: Never` dans le `deployment.yaml` et toujours faire `k3d image import` avant de déployer.

### Problème 3 : Mauvais interpréteur Python pour Ansible
**Erreur :** `Failed to import the required Python library (kubernetes)`  
**Solution :** Ajouter `-e "ansible_python_interpreter=/usr/bin/python3"` à la commande ansible-playbook.

### Problème 4 : Port 8080 déjà utilisé par Mario
**Solution :** Utiliser le port `8081` pour nginx-custom ou tuer le port-forward existant :
```bash
pkill -f "port-forward svc/mario"
```

---

## 👩‍💻 Auteur

**zinebbouayad** — Atelier réalisé dans GitHub Codespaces avec K3d v5.8.3, Packer v1.15.1, Ansible 9.2.0 et Kubernetes v1.31.5.
