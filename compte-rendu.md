# Compte-rendu — Mini-Projet 1 : Application web à deux niveaux sur Kubernetes

**Binôme :** Ferid & Saad | **Namespace :** `feridsaad` | **Date :** Octobre 2026

---

## 1. Schéma d'architecture

![Architecture Kubernetes – Namespace feridsaad](C:\Users\fersd\.gemini\antigravity\brain\c7bafe84-2e04-4b6c-bb48-f5b7973b7641\architecture_k8s_1792324041190.png)

L'architecture déployée comprend :
- Un **Ingress Traefik** (point d'entrée unique, IP `192.168.56.10`) avec deux règles par hôte
- Deux **Services ClusterIP** (`api-svc` port 80→9898, `frontend-svc` port 80→80)
- Un **Deployment API** (3 réplicas, `podinfo:6.5.4`, `tier: backend`)
- Un **Deployment Frontend** (2 réplicas, `nginx:1.27-alpine`, `tier: web`)
- Une communication DNS interne via `http://api-svc` du frontend vers l'API

---

## 2. Réponses aux questions de chaque étape

### Étape 0 — Pourquoi isoler dans un namespace ?

Un namespace est un **espace de noms virtuel** au sein du cluster Kubernetes. L'isolation apporte quatre bénéfices :

1. **Pas de conflit de noms** : deux équipes peuvent avoir un Deployment nommé `api` sans collision ;
2. **Contrôle des accès (RBAC)** : les droits peuvent être restreints par namespace ;
3. **Quotas de ressources** (`ResourceQuota`) : limite CPU/mémoire par équipe/projet ;
4. **Nettoyage simplifié** : `kubectl delete namespace feridsaad` supprime toutes les ressources du projet en une seule commande.

---

### Étape 1 — Sur quels nœuds les Pods sont-ils répartis ? Qui décide ?

C'est le **Scheduler Kubernetes** (`kube-scheduler`) qui prend cette décision automatiquement. Il évalue les nœuds disponibles selon :
- **Filtrage** : ressources suffisantes (CPU, RAM), taints/tolerations, affinités
- **Scoring** : meilleure utilisation des ressources, spreading des Pods pour la haute disponibilité

Par défaut, le scheduler tend à **répartir équitablement** les Pods entre les nœuds pour éviter un point de défaillance unique.

---

### Étape 2 — Que se passerait-il si les deux Deployments avaient les mêmes labels ?

Si `frontend` portait aussi `app=api, tier=backend` :
- Le `selector` de `api-svc` sélectionnerait également les Pods frontend → **trafic envoyé au mauvais conteneur** (nginx au lieu de podinfo)
- Les Deployments se disputeraient la propriété des mêmes Pods
- Les `readinessProbe` échoueraient (nginx ne répond pas sur `/readyz`)

**Les labels doivent être uniques** par Deployment pour que les sélecteurs de Service fonctionnent correctement.

---

### Étape 3 — Nom DNS complet et utilité de `get endpoints`

**Nom DNS complet d'un Service :**
```
<nom-service>.<namespace>.svc.cluster.local
→ api-svc.feridsaad.svc.cluster.local
```
Le suffixe `.svc.cluster.local` est résolu automatiquement par **CoreDNS**. Depuis le même namespace, `api-svc` suffit.

**Utilité de `kubectl get endpoints` :**
`get endpoints` liste les adresses IP:port des Pods effectivement sélectionnés par un Service. Si la colonne `ENDPOINTS` affiche `<none>`, cela signifie que **le sélecteur du Service ne correspond à aucun Pod** — c'est le premier indice d'un problème de label ou de santé des Pods. C'est l'outil de diagnostic n°1 pour les problèmes réseau.

---

### Étape 4 — Pourquoi préférer Ingress à deux NodePort ?

| Critère | NodePort (par app) | Ingress (centralisé) |
|---|---|---|
| Port exposé | Un port dédié 30000–32767 par app | Port 80/443 unique |
| URLs | `http://IP:31234` et `http://IP:31567` | `http://web.k3s.local` et `http://api.k3s.local` |
| TLS/HTTPS | À gérer sur chaque app | Terminaison TLS centralisée |
| Routage | Par port uniquement | Par hôte **ET** par chemin (`/api/`, `/`) |
| Maintenance | Un port/Service → difficile à gérer | Un seul contrôleur pour N applications |

L'Ingress est la solution standard en production : **un seul point d'entrée**, routage basé sur le nom d'hôte, TLS gérable centralement.

---

### Étape 5 — Rolling Update : ReplicaSets et rollback

**Après la mise à jour (6.5.4 → 6.6.0), il existe 3 ReplicaSets :**

```
api-5fd8c99f6b   DESIRED=3, READY=3  ← NOUVEAU (v6.6.0), actif
api-6cfc44d4d5   DESIRED=0, READY=0  ← ANCIEN (v6.5.4), conservé à 0 réplica
api-6b7b4d8598   DESIRED=0, READY=0  ← révision initiale
```

L'**ancien ReplicaSet est conservé** (à 0 réplica) car Kubernetes maintient un historique des révisions (configurable via `revisionHistoryLimit`, défaut = 10). Cela permet un **rollback instantané** sans rebuild d'image : Kubernetes réactive simplement l'ancien ReplicaSet.

**Commande de rollback :**
```bash
kubectl rollout undo deployment/api -n feridsaad
# Pour une révision précise :
kubectl rollout undo deployment/api --to-revision=2 -n feridsaad
```

---

### Étape 6 — Panne et diagnostic

**Incident observé :** Pods `Evicted` lors du rolling update sur VirtualBox (pression mémoire).

**Symptômes → Cause → Correction :** voir captures ci-dessous et section 3.

---

## 3. Captures prouvant chaque exigence

### E1–E7 : État du cluster

![kubectl get all,ingress — état propre du cluster](C:\Users\fersd\.gemini\antigravity\brain\c7bafe84-2e04-4b6c-bb48-f5b7973b7641\capture_get_all_1792324050224.png)

> **Lecture :** 3 Pods API `Running` (E2), 2 Pods frontend `Running` (E3), 2 Services `ClusterIP` (E5), Ingress Traefik sur `web.k3s.local` et `api.k3s.local` avec IP `192.168.56.10` (E7).

---

### E6 : Communication DNS interne + E7 : Accès Ingress

![E6 DNS interne et E7 Ingress](C:\Users\fersd\.gemini\antigravity\brain\c7bafe84-2e04-4b6c-bb48-f5b7973b7641\capture_dns_ingress_1792324112485.png)

> **E6 prouvé :** `kubectl exec deploy/frontend -- wget -qO- http://api-svc/` retourne `{"version":"6.5.4"}` — le frontend joint l'API par son nom DNS sans adresse IP.
>
> **E7 prouvé :** `curl -H "Host: api.k3s.local" http://192.168.56.10` retourne la réponse de l'API via l'Ingress Traefik.
>
> **Endpoints** : `api-svc` liste 3 IPs:9898 et `frontend-svc` liste 2 IPs:80 — les Services pointent vers les bons Pods.

---

### E9 : Rolling Update sans coupure

![E9 Rolling Update 6.5.4 → 6.6.0 → rollback](C:\Users\fersd\.gemini\antigravity\brain\c7bafe84-2e04-4b6c-bb48-f5b7973b7641\capture_rolling_update_1792324123068.png)

> **Lecture :** La boucle de 20 requêtes montre la **transition progressive** entre `6.5.4` et `6.6.0` sans aucune interruption de service. Les anciens Pods restent actifs (`6.5.4`) jusqu'à ce que les nouveaux soient `Ready` (`maxUnavailable: 0`). L'ancien ReplicaSet est conservé à 0 réplica pour le rollback. Le `rollout undo` remet l'API à `6.5.4` avec succès.

---

### E10 : Diagnostic de panne

![E10 Diagnostic panne — Pods Evicted](C:\Users\fersd\.gemini\antigravity\brain\c7bafe84-2e04-4b6c-bb48-f5b7973b7641\capture_diagnostic_1792324142361.png)

> **Démarche :**
> 1. `kubectl get pods` → Pods en `Evicted` / `Error`
> 2. `kubectl get endpoints` → endpoints toujours présents (les Pods v6.5.4 survivants servaient encore le trafic)
> 3. `kubectl describe node` → `Reason=Evicted, Message=low on resource: memory`
> 4. **Correction :** `kubectl delete pods --field-selector=status.phase=Failed` + `maxSurge: 0` dans le manifeste

---

## 4. Difficultés rencontrées et enseignements

### Difficulté 1 — Évictions de Pods sur VirtualBox

**Problème :** Lors du déploiement initial et du rolling update, plusieurs Pods se retrouvaient en état `Evicted` ou `Error`.

**Cause :** La VM VirtualBox dispose de peu de RAM. Le `maxSurge: 1` créait un Pod supplémentaire pendant la mise à jour, ce qui saturait la mémoire et déclenchait des évictions par le `kubelet`.

**Solution :** Passer à `maxSurge: 0` (mise à jour en place, un Pod supprimé avant qu'un nouveau soit créé) et réduire les `resources.limits` mémoire à 32Mi pour le frontend.

**Enseignement :** Les stratégies de rolling update doivent être **adaptées aux ressources disponibles** du cluster. `maxSurge: 0` est plus sûr sur un cluster contraint, au prix d'une capacité temporairement réduite.

---

### Difficulté 2 — Fichier `hosts` sur Windows

**Problème :** Le fichier `C:\Windows\System32\drivers\etc\hosts` requiert des droits administrateur pour être modifié.

**Solution alternative :** Utiliser l'en-tête `Host` avec curl pendant les tests : `curl.exe -H "Host: api.k3s.local" http://192.168.56.10`, ce qui évite la modification du fichier système.

---

### Difficulté 3 — Readiness Probe et timing de démarrage

**Problème :** Sans `readinessProbe`, Kubernetes envoyait du trafic à des Pods non encore opérationnels, provoquant des erreurs 502 lors du rolling update.

**Solution :** La `readinessProbe` sur `/readyz` (podinfo) garantit qu'un Pod ne reçoit du trafic **que lorsqu'il est prêt**, ce qui élimine les erreurs transitoires.

---

### Enseignements clés

| Concept | Enseignement |
|---|---|
| **Namespace** | Isolation propre par équipe/projet, nettoyage facilité |
| **Labels & sélecteurs** | Doivent être uniques et cohérents entre Deployment et Service |
| **DNS interne** | `api-svc.feridsaad.svc.cluster.local` — pas d'IP à gérer |
| **`get endpoints`** | Premier réflexe de diagnostic réseau en K8s |
| **Ingress vs NodePort** | Ingress = un point d'entrée, routage par hôte, TLS centralisé |
| **Rolling Update** | `maxUnavailable: 0` + `readinessProbe` = zéro coupure |
| **Rollback** | Instantané grâce à la conservation des anciens ReplicaSets |
| **Evictions** | Signe de pression mémoire → réduire `maxSurge` et les limites |
