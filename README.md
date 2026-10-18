# Mini-Projet 1 — Application web à deux niveaux sur k3s

> **Binôme :** Ferid & Saad — Namespace `feridsaad`

## Architecture déployée

```
         Utilisateur (navigateur / curl)
                     |
              Ingress (Traefik)
         web.k3s.local | api.k3s.local
             /                  \
  Service frontend-svc      Service api-svc
       (ClusterIP)              (ClusterIP)
            |                        |
  Deployment frontend       Deployment api
    2 réplicas (nginx)      3 réplicas (podinfo)
```

## Contenu du dépôt

```
manifests/
├── 00-namespace.yaml          # Namespace feridsaad (E1)
├── 10-api-deployment.yaml     # API podinfo 6.5.4 – 3 réplicas (E2, E4)
├── 20-frontend-deployment.yaml # Frontend nginx – 2 réplicas + ConfigMap (E3, E4)
├── 30-services.yaml           # Services ClusterIP api-svc et frontend-svc (E5)
└── 40-ingress.yaml            # Ingress Traefik (web.k3s.local, api.k3s.local) (E7)
```

---

## Déploiement complet (ordre à respecter)

### 1. Cloner le dépôt et se positionner

```bash
git clone <URL_DU_DEPOT>
cd Mini-projet-1
```

### 2. Créer le namespace et le définir par défaut

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=feridsaad
```

### 3. Appliquer tous les manifestes

```bash
kubectl apply -f manifests/
```

### 4. Vérifier que tout est en ordre

```bash
# Voir tous les objets dans le namespace
kubectl get all,ingress,configmap

# Vérifier les Pods et leur répartition sur les nœuds
kubectl get pods -o wide

# Vérifier les Endpoints (preuves que les Services pointent bien vers les Pods)
kubectl get endpoints
```

Sortie attendue :
```
NAME                           READY   STATUS    RESTARTS   AGE   NODE
pod/api-xxx-yyy               1/1     Running   0          30s   node1
pod/api-xxx-zzz               1/1     Running   0          30s   node2
pod/api-xxx-www               1/1     Running   0          30s   node1
pod/frontend-xxx-aaa          1/1     Running   0          28s   node2
pod/frontend-xxx-bbb          1/1     Running   0          28s   node1
```

---

## Accès depuis Windows (Ingress)

Ajouter dans `C:\Windows\System32\drivers\etc\hosts` (en tant qu'administrateur) :

```
<IP_DU_NOEUD_K3S>  web.k3s.local  api.k3s.local
```

> Trouver l'IP du nœud : `kubectl get nodes -o wide`

Tester avec curl :

```powershell
# API (renvoie du JSON avec la version)
curl.exe -s http://api.k3s.local | ConvertFrom-Json

# Frontend
curl.exe http://web.k3s.local

# Tester avec l'en-tête Host (sans modifier hosts)
curl.exe -H "Host: api.k3s.local" http://<IP_DU_NOEUD>
curl.exe -H "Host: web.k3s.local" http://<IP_DU_NOEUD>
```

---

## Test de communication interne frontend → API (E6)

```bash
# Entrer dans un Pod du frontend
kubectl exec -it deploy/frontend -- sh

# Depuis le Pod, appeler l'API par son nom DNS (sans IP)
wget -qO- http://api-svc/
# ou avec le nom complet :
wget -qO- http://api-svc.feridsaad.svc.cluster.local/
```

---

## Mise à jour sans coupure — Rolling Update (E9)

### Terminal 1 — Boucle de surveillance

```bash
# Linux/macOS
while true; do curl -s http://api.k3s.local/version | grep version; sleep 0.5; done

# Windows PowerShell
while ($true) { (curl.exe -s http://api.k3s.local/version | ConvertFrom-Json).version; Start-Sleep -Milliseconds 500 }
```

### Terminal 2 — Mise à jour de l'image

Modifier `manifests/10-api-deployment.yaml` : remplacer `6.5.4` par `6.6.0`

```bash
kubectl apply -f manifests/10-api-deployment.yaml
kubectl rollout status deployment/api   # Suivre la progression
kubectl get replicasets                 # Voir les deux ReplicaSets coexister
```

### Rollback (retour à la version précédente)

```bash
kubectl rollout undo deployment/api
kubectl rollout status deployment/api
kubectl get replicasets
```

---

## Diagnostic de panne (E10)

Démarche méthodique (ordre des investigations) :

```bash
# 1. Vue d'ensemble — y a-t-il des Pods en erreur ?
kubectl get pods

# 2. Détail d'un Pod suspect (events + état)
kubectl describe pod <nom-du-pod>

# 3. Logs du conteneur
kubectl logs <nom-du-pod>
kubectl logs <nom-du-pod> --previous   # Si le Pod a redémarré

# 4. Le Service envoie-t-il du trafic aux bons Pods ?
kubectl get endpoints api-svc
kubectl get endpoints frontend-svc

# 5. Vérifier les sélecteurs du Service vs labels des Pods
kubectl get svc api-svc -o yaml | grep -A5 selector
kubectl get pods --show-labels

# 6. Tester la connectivité depuis l'intérieur du cluster
kubectl run test --rm -it --image=busybox -- wget -qO- http://api-svc/
```

---

## Exigences satisfaites

| N°  | Exigence                                      | Fichier                          |
|-----|-----------------------------------------------|----------------------------------|
| E1  | Namespace dédié `feridsaad`                   | `00-namespace.yaml`              |
| E2  | API : 3 réplicas, image versionnée (6.5.4)    | `10-api-deployment.yaml`         |
| E3  | Frontend : 2 réplicas                         | `20-frontend-deployment.yaml`    |
| E4  | Labels cohérents `app` + `tier`               | Les deux Deployments             |
| E5  | Services type ClusterIP                       | `30-services.yaml`               |
| E6  | Communication frontend→API par nom DNS        | ConfigMap nginx + `api-svc`      |
| E7  | Ingress Traefik sur deux hôtes                | `40-ingress.yaml`                |
| E8  | Tout déclaratif, versionné dans Git           | Ce dépôt                         |
| E9  | Rolling update sans coupure (maxUnavailable=0)| `10-api-deployment.yaml`         |
| E10 | Démarche de diagnostic documentée            | Compte-rendu + section ci-dessus |
