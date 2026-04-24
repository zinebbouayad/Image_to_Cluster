------------------------------------------------------------------------------------------------------
ATELIER FROM IMAGE TO CLUSTER
------------------------------------------------------------------------------------------------------
L’idée en 30 secondes : Cet atelier consiste à **industrialiser le cycle de vie d’une application** simple en construisant une **image applicative Nginx** personnalisée avec **Packer**, puis en déployant automatiquement cette application sur un **cluster Kubernetes** léger (K3d) à l’aide d’**Ansible**, le tout dans un environnement reproductible via **GitHub Codespaces**.
L’objectif est de comprendre comment des outils d’Infrastructure as Code permettent de passer d’un artefact applicatif maîtrisé à un déploiement cohérent et automatisé sur une plateforme d’exécution.
  
-------------------------------------------------------------------------------------------------------
Séquence 1 : Codespace de Github
-------------------------------------------------------------------------------------------------------
Objectif : Création d'un Codespace Github  
Difficulté : Très facile (~5 minutes)
-------------------------------------------------------------------------------------------------------
**Faites un Fork de ce projet**. Si besion, voici une vidéo d'accompagnement pour vous aider dans les "Forks" : [Forker ce projet](https://youtu.be/p33-7XQ29zQ) 
  
Ensuite depuis l'onglet [CODE] de votre nouveau Repository, **ouvrez un Codespace Github**.
  
---------------------------------------------------
Séquence 2 : Création du cluster Kubernetes K3d
---------------------------------------------------
Objectif : Créer votre cluster Kubernetes K3d  
Difficulté : Simple (~5 minutes)
---------------------------------------------------
Vous allez dans cette séquence mettre en place un cluster Kubernetes K3d contenant un master et 2 workers.  
Dans le terminal du Codespace copier/coller les codes ci-dessous etape par étape :  

**Création du cluster K3d**  
```
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```
```
k3d cluster create lab \
  --servers 1 \
  --agents 2
```
**vérification du cluster**  
```
kubectl get nodes
```
**Déploiement d'une application (Docker Mario)**  
```
kubectl create deployment mario --image=sevenajay/mario
kubectl expose deployment mario --type=NodePort --port=80
kubectl get svc
```
**Forward du port 80**  
```
kubectl port-forward svc/mario 8080:80 >/tmp/mario.log 2>&1 &
```
**Réccupération de l'URL de l'application Mario** 
Votre application Mario est déployée sur le cluster K3d. Pour obtenir votre URL cliquez sur l'onglet **[PORTS]** dans votre Codespace et rendez public votre port **8080** (Visibilité du port).
Ouvrez l'URL dans votre navigateur et jouer !

---------------------------------------------------
Séquence 3 : Exercice
---------------------------------------------------
Objectif : Customisez un image Docker avec Packer et déploiement sur K3d via Ansible
Difficulté : Moyen/Difficile (~2h)
---------------------------------------------------  
Votre mission (si vous l'acceptez) : Créez une **image applicative customisée à l'aide de Packer** (Image de base Nginx embarquant le fichier index.html présent à la racine de ce Repository), puis déployer cette image customisée sur votre **cluster K3d** via **Ansible**, le tout toujours dans **GitHub Codespace**.  

**Architecture cible :** Ci-dessous, l'architecture cible souhaitée.   
  
![Screenshot Actions](Architecture_cible.png)   
  
---------------------------------------------------  
## Processus de travail (résumé)

1. Installation du cluster Kubernetes K3d (Séquence 1)
2. Installation de Packer et Ansible
3. Build de l'image customisée (Nginx + index.html)
4. Import de l'image dans K3d
5. Déploiement du service dans K3d via Ansible
6. Ouverture des ports et vérification du fonctionnement

---------------------------------------------------
# Séquence 3 — From Image to Cluster

## Structure du projet

```
.                          ← racine du repo (contient index.html)
├── packer/
│   └── nginx.pkr.hcl      ← template Packer (build image Nginx + index.html)
├── ansible/
│   ├── deploy.yml         ← playbook de déploiement K3d
│   ├── inventory.ini      ← inventaire Ansible (localhost)
│   └── requirements.yml   ← collection kubernetes.core
└── run_sequence3.sh       ← script tout-en-un
```

## Exécution rapide (script tout-en-un)

```bash
chmod +x run_sequence3.sh
./run_sequence3.sh
```

## Exécution étape par étape

### 1. Installer les outils

```bash
# Packer
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y packer

# Ansible + dépendances Python
sudo apt-get install -y ansible python3-pip
pip install kubernetes
ansible-galaxy collection install -r ansible/requirements.yml
```

### 2. Créer le cluster K3d 

```bash
k3d cluster create lab --servers 1 --agents 2
kubectl get nodes
```

### 3. Builder l'image avec Packer

```bash
cd packer/
packer init .
packer build -force .
# → génère nginx-custom.tar dans packer/
ls -lh nginx-custom.tar
```

### 4. Déployer avec Ansible

```bash
cd ansible/
ansible-playbook deploy.yml \
  -i inventory.ini \
  -e "image_tar=$(pwd)/../packer/nginx-custom.tar" \
  -v
```

### 5. Vérifier le déploiement

```bash
kubectl get deployments,pods,svc -l app=nginx-custom
curl http://localhost:8080
```

### 6. Exposer dans Codespace

Onglet **[PORTS]** → port `8080` → **Visibilité : Public** → ouvrir l'URL.

## Ce que fait chaque outil

| Outil | Rôle |
|-------|------|
| **Packer** | Build une image Docker `nginx-custom:latest` à partir de `nginx:alpine` en y copiant `index.html` |
| **k3d image import** | Charge l'image tar dans le registre interne du cluster K3d sans passer par Docker Hub |
| **Ansible** | Automatise l'import, le déploiement (Deployment + Service), l'attente de readiness et le port-forward |

## Dépannage

```bash
# Voir les logs du pod
kubectl logs -l app=nginx-custom

# Voir le log du port-forward
cat /tmp/nginx-pf.log

# Relancer manuellement le port-forward
kubectl port-forward svc/nginx-custom 8080:80 >/tmp/nginx-pf.log 2>&1 &
```

--------------------------------------------------
---------------------------------------------------
Séquence 4 : Documentation  
Difficulté : Facile (~30 minutes)
---------------------------------------------------
**Complétez et documentez ce fichier README.md** pour nous expliquer comment utiliser votre solution.  
Faites preuve de pédagogie et soyez clair dans vos expliquations et processus de travail.  
   
---------------------------------------------------
Evaluation
---------------------------------------------------
Cet atelier, **noté sur 20 points**, est évalué sur la base du barème suivant :  
- Repository exécutable sans erreur majeure (4 points)
- Fonctionnement conforme au scénario annoncé (4 points)
- Degré d'automatisation du projet (utilisation de Makefile ? script ? ...) (4 points)
- Qualité du Readme (lisibilité, erreur, ...) (4 points)
- Processus travail (quantité de commits, cohérence globale, interventions externes, ...) (4 points) 


