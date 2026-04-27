# TP #5 — Environnements dev/staging/prod avec namespaces & RBAC

## Structure du projet

```
k8s/
├── base/                  # Manifests communs (Kustomize)
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
├── overlays/
│   ├── dev/               # Overlay dev (baseline PSS, quotas serrés)
│   ├── staging/           # Overlay staging (baseline PSS, 2 replicas)
│   └── prod/              # Overlay prod (restricted PSS, NetworkPolicies, 3 replicas)
└── rbac/                  # Roles et RoleBindings
```

## Commandes d'installation

### 1. Déployer les 3 environnements

```bash
kubectl apply -k k8s/overlays/dev
kubectl apply -k k8s/overlays/staging
kubectl apply -k k8s/overlays/prod
```

### 2. Appliquer les RBAC

Les Roles sont namespace-scoped, il faut les appliquer dans chaque namespace concerné :

```bash
# developer-role dans dev et staging
kubectl apply -f k8s/rbac/developer-role.yaml -n app-dev
kubectl apply -f k8s/rbac/developer-role.yaml -n app-staging
kubectl apply -f k8s/rbac/developer-binding-dev.yaml
kubectl apply -f k8s/rbac/developer-binding-staging.yaml

# qa-role dans staging
kubectl apply -f k8s/rbac/qa-role.yaml -n app-staging
kubectl apply -f k8s/rbac/qa-binding-staging.yaml

# prod-deployer dans prod
kubectl apply -f k8s/rbac/prod-deployer-role.yaml
kubectl apply -f k8s/rbac/prod-deployer-binding.yaml
```

### 3. Simuler les utilisateurs (contexts kubectl)

```bash
# Créer des clés et CSR pour chaque utilisateur
openssl genrsa -out dev-user.key 2048
openssl req -new -key dev-user.key -out dev-user.csr -subj "/CN=dev-user"

openssl genrsa -out qa-user.key 2048
openssl req -new -key qa-user.key -out qa-user.csr -subj "/CN=qa-user"

openssl genrsa -out prod-deployer.key 2048
openssl req -new -key prod-deployer.key -out prod-deployer.csr -subj "/CN=prod-deployer"

# Signer avec le CA du cluster (Minikube)
CA_CERT=~/.minikube/ca.crt
CA_KEY=~/.minikube/ca.key

openssl x509 -req -in dev-user.csr -CA $CA_CERT -CAkey $CA_KEY -CAcreateserial -out dev-user.crt -days 365
openssl x509 -req -in qa-user.csr -CA $CA_CERT -CAkey $CA_KEY -CAcreateserial -out qa-user.crt -days 365
openssl x509 -req -in prod-deployer.csr -CA $CA_CERT -CAkey $CA_KEY -CAcreateserial -out prod-deployer.crt -days 365

# Configurer les credentials et contexts
kubectl config set-credentials dev-user --client-certificate=dev-user.crt --client-key=dev-user.key
kubectl config set-credentials qa-user --client-certificate=qa-user.crt --client-key=qa-user.key
kubectl config set-credentials prod-deployer --client-certificate=prod-deployer.crt --client-key=prod-deployer.key

CLUSTER_NAME=$(kubectl config view -o jsonpath='{.clusters[0].name}')
kubectl config set-context dev-user@app-cluster --cluster=$CLUSTER_NAME --user=dev-user
kubectl config set-context qa-user@app-cluster --cluster=$CLUSTER_NAME --user=qa-user
kubectl config set-context prod-deployer@app-cluster --cluster=$CLUSTER_NAME --user=prod-deployer
```

## Commandes de test (preuves)

### Vérifier les namespaces et labels

```bash
kubectl get ns --show-labels | grep app-
```

### Vérifier les déploiements

```bash
kubectl get all -n app-dev
kubectl get all -n app-staging
kubectl get all -n app-prod
```

### Tester l'app (réponse par environnement)

```bash
kubectl port-forward svc/demo-app -n app-dev 8081:80 &
curl http://localhost:8081
# => hello from dev

kubectl port-forward svc/demo-app -n app-staging 8082:80 &
curl http://localhost:8082
# => hello from staging

kubectl port-forward svc/demo-app -n app-prod 8083:80 &
curl http://localhost:8083
# => hello from prod
```

### Tester RBAC

```bash
# dev-user peut déployer en dev
kubectl --context dev-user@app-cluster auth can-i create deployment -n app-dev
# => yes

# dev-user peut lire les logs en staging
kubectl --context dev-user@app-cluster auth can-i get pods/log -n app-staging
# => yes

# dev-user est BLOQUÉ en prod
kubectl --context dev-user@app-cluster auth can-i get pods -n app-prod
# => no

# qa-user peut lire en staging mais pas créer
kubectl --context qa-user@app-cluster auth can-i get pods -n app-staging
# => yes
kubectl --context qa-user@app-cluster auth can-i create deployment -n app-staging
# => no

# prod-deployer peut patcher les deployments en prod
kubectl --context prod-deployer@app-cluster auth can-i patch deployment -n app-prod
# => yes
```

### Tester les quotas (dépassement en dev)

```bash
# Tenter de créer un pod qui dépasse le quota
kubectl run quota-test --image=nginx --restart=Never -n app-dev \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","resources":{"requests":{"cpu":"2","memory":"1Gi"},"limits":{"cpu":"2","memory":"1Gi"}}}]}}'
# => Error : exceeded quota
kubectl delete pod quota-test -n app-dev --ignore-not-found
```

### Tester PSS en prod (pod non conforme refusé)

```bash
kubectl run pss-test --image=nginx --restart=Never -n app-prod \
  --overrides='{"spec":{"containers":[{"name":"nginx","image":"nginx","securityContext":{"privileged":true}}]}}'
# => Error : violates PodSecurity "restricted"
```

### Tester NetworkPolicy (isolation prod)

```bash
# Depuis un pod en dev, tenter d'appeler le service en prod
kubectl run nettest --image=busybox --restart=Never -n app-dev -- wget -qO- --timeout=3 http://demo-app.app-prod.svc.cluster.local
kubectl logs nettest -n app-dev
# => timeout (bloqué par la NetworkPolicy)
kubectl delete pod nettest -n app-dev --ignore-not-found
```

## Tableau RBAC — Qui a le droit de faire quoi

```
Persona          | app-dev          | app-staging      | app-prod
-----------------|------------------|------------------|------------------
platform-admin   | Tout (admin)     | Tout (admin)     | Tout (admin)
dev-user         | CRUD deploy/svc  | CRUD deploy/svc  | AUCUN accès
                 | pods/log, exec   | pods/log, exec   |
                 | Pas de secrets   | Pas de secrets   |
qa-user          | Aucun accès      | Lecture pods/svc  | Aucun accès
                 |                  | pods/log          |
                 |                  | patch deploy      |
prod-deployer    | Aucun accès      | Aucun accès      | get/patch deploy
                 |                  |                  | CRUD svc/ingress
                 |                  |                  | pods/log (lecture)
```

Note : aucun persona (sauf platform-admin) n'a accès aux Secrets. En entreprise, les secrets sont injectés via un opérateur externe (Vault, Sealed Secrets, External Secrets Operator).

## Checklist bonnes pratiques

- [x] **Séparation des environnements** : 3 namespaces avec labels standards (`environment`, `owner`, `part-of`)
- [x] **RBAC** : principe du moindre privilège, pas d'accès secrets pour les devs, prod verrouillée
- [x] **Secrets** : non exposés via RBAC. En production, utiliser un opérateur (Vault, Sealed Secrets) plutôt que des Secrets K8s natifs
- [x] **Quotas / LimitRange** : quotas CPU/RAM/pods par namespace, defaults et max par container
- [x] **Pod Security Standards** : `baseline` en dev/staging, `restricted` en prod (runAsNonRoot, drop ALL capabilities, readOnlyRootFilesystem)
- [x] **NetworkPolicies** : default deny en prod, autorisation explicite ingress-nginx + DNS egress
- [x] **Stratégie de release** : RollingUpdate avec maxSurge=1, maxUnavailable=0. Rollback via `kubectl rollout undo`
- [x] **Politique images** : utiliser un registry privé, scanner les images (Trivy), signer (cosign/Notary), interdire `latest` tag
