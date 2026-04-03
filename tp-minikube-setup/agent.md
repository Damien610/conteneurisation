# TP — Setup Minikube

# https://sysentive.link/tp-minikube-setup

## Objectifs

- Installer Minikube et kubectl pour exécuter Kubernetes en local.
- Démarrer un cluster local avec le driver Docker.
- Déployer un premier conteneur (Deployment) et l’exposer.
- Vérifier l’état via les commandes de base (nodes, pods, services, logs).

## Prérequis

- Docker installé et démarré.

---

## 1) Installer les outils

### 1.1 Installer kubectl

Installez kubectl pour votre système d’exploitation.

Vérifiez :

- `kubectl version --client`

### 1.2 Installer Minikube

Installez Minikube pour votre système d’exploitation.

Vérifiez :

- `minikube version`

### 1.3 Vérifier Docker (driver unique)

Dans ce TP, vous utiliserez uniquement le driver Docker.

- Assurez-vous que Docker est bien démarré.
- Vérifiez que la commande `docker version` fonctionne.

---

## 2) Démarrer un cluster Minikube

### 2.1 Démarrer

Démarrez Minikube avec le driver Docker :

- `minikube start --driver=docker`

Attendez que le cluster soit “Running”.

### 2.2 Vérifier le contexte kubectl

- `kubectl config current-context`
- `kubectl cluster-info`
- `kubectl get nodes`

#### Questions

1. Quel est le nom du contexte courant (context) ?
2. Combien de nœuds voyez-vous ?

---

## 3) Déployer un premier conteneur (nginx)

### 3.1 Créer un namespace

- `kubectl create namespace tp-minikube`
- `kubectl get ns`

Pour la suite, ajoutez `-n tp-minikube` à vos commandes.

### 3.2 Écrire le manifest Deployment (à vous de jouer)

Créez un fichier `deployment.yaml` qui déploie un pod nginx.

Contraintes attendues :

- `kind: Deployment` (API version adaptée)
- Nom du deployment : `web-nginx`
- `replicas: 1`
- Labels cohérents (par exemple `app: web-nginx`) sur :
    - `metadata.labels`
    - `spec.selector.matchLabels`
    - `spec.template.metadata.labels`
- Un conteneur :
    - `name: nginx`
    - `image: nginx:1.27`
    - Port conteneur 80 déclaré

Appliquez :

- `kubectl apply -f deployment.yaml -n tp-minikube`

Vérifiez :

- `kubectl get deployments -n tp-minikube`
- `kubectl get pods -n tp-minikube`

### 3.3 Inspecter le pod

- `kubectl describe pod -n tp-minikube -l app=web-nginx`

#### Questions

1. Quelle image exacte est utilisée ?
2. Quel évènement (Events) confirme le téléchargement et le démarrage ?

---

## 4) Exposer l’application

### 4.1 Écrire le manifest Service (à vous de jouer)

Créez un fichier `service.yaml` pour exposer nginx.

Contraintes attendues (minimum) :

- `kind: Service` (API version adaptée)
- Nom du service : `web-nginx-svc`
- Le Service doit sélectionner les pods du Deployment via le label `app: web-nginx`
- Exposer le port 80 (HTTP) vers le `targetPort` 80
- Type : `ClusterIP`

Appliquez :

- `kubectl apply -f service.yaml -n tp-minikube`

Vérifiez :

- `kubectl get svc -n tp-minikube`

### 4.2 Accéder au service depuis votre machine

Récupérez une URL d’accès avec Minikube :

- `minikube service web-nginx-svc -n tp-minikube --url`

Ouvrez l’URL renvoyée dans votre navigateur.

#### Questions

1. À quoi sert un Service dans Kubernetes ?
2. Pourquoi le Service n’expose-t-il pas automatiquement une IP publique en local ?

---

## 5) Observer, diagnostiquer, itérer

### 5.1 Logs

- `kubectl logs -n tp-minikube -l app=web-nginx`

### 5.2 Rollout / mise à jour d’image

Modifiez l’image en `nginx:1.26`, puis mettez à jour le Deployment :

- `kubectl set image -n tp-minikube deployment/web-nginx nginx=nginx:1.26`

Suivez le déploiement :

- `kubectl rollout status -n tp-minikube deployment/web-nginx`
- `kubectl get pods -n tp-minikube -w`

#### Questions

1. Que se passe-t-il au niveau des pods lors d’un changement d’image ?
2. Qu’est-ce qu’un “rollout” et pourquoi est-ce utile ?

---

## 6) Bonus — Dashboard Kubernetes

- `minikube dashboard`

Repérez :

- le namespace `tp-minikube`
- le Deployment et le Service
- l’état des pods

---

## 7) Nettoyage

Supprimez les ressources :

- `kubectl delete namespace tp-minikube`

Arrêtez Minikube :

- `minikube stop`

Supprimez le cluster (si besoin) :

- `minikube delete`

---

## Livrables attendus

- Les fichiers `deployment.yaml` et `service.yaml`.
- Un mini compte-rendu (8–12 lignes) :
    - les commandes de vérification (nodes/pods/svc)
    - comment vous avez exposé le service
    - 2 difficultés rencontrées + comment vous les avez résolues