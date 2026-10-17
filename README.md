# Mini-projet 1 : application web à deux niveaux sur k3s

## Contenu

* `manifests/00-namespace.yaml` : namespace
* `manifests/10-api-deployment.yaml` : API podinfo (3 réplicas, version 6.5.4)
* `manifests/20-frontend-deployment.yaml` : frontend nginx (2 réplicas)
* `manifests/30-services.yaml` : Services ClusterIP `api-svc` et `frontend-svc`
* `manifests/40-ingress.yaml` : Ingress Traefik (`web.k3s.local`, `api.k3s.local`)

## Déploiement

```bash
kubectl apply -f manifests/00-namespace.yaml
kubectl config set-context --current --namespace=feridsaad
kubectl apply -f manifests/
kubectl get all,ingress
```

## Accès

Ajouter dans `C:\\Windows\\System32\\drivers\\etc\\hosts` :
`<IP\_NOEUD> web.k3s.local api.k3s.local`
puis : `curl.exe http://api.k3s.local`

## Mise à jour / rollback

```bash
# modifier l'image dans 10-api-deployment.yaml (ex. 6.5.4 -> 6.6.0), puis :
kubectl apply -f manifests/10-api-deployment.yaml
kubectl rollout status deployment/api
kubectl rollout undo deployment/api
```

