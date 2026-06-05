```bash
# Vérifier que Docker est actif
docker info

# Construire l'image multi-stage
docker build -t taskflow:1.0.0 .

# (Optionnel) Tester l'image localement
docker run --rm -d -p 8080:80 --name taskflow-test taskflow:1.0.0
curl http://localhost:8080
docker stop taskflow-test
```

```bash
# Vérifier les prérequis
k3d version
kubectl version --client

# Créer le cluster avec mapping de port 8080 → 80
k3d cluster create taskflow --port "8080:80@loadbalancer"

# Vérifier que le nœud est Ready
kubectl get nodes

# Importer l'image locale dans k3d
k3d image import taskflow:1.0.0 -c taskflow
```


> Les 6 fichiers ont été créés dans le dossier `k8s/`.

| Fichier | Ressource |
|---|---|
| `k8s/namespace.yaml` | Namespace `taskflow` |
| `k8s/configmap.yaml` | ConfigMap `taskflow-config` |
| `k8s/deployment.yaml` | Deployment (2 replicas, image `taskflow:1.0.0`) |
| `k8s/service.yaml` | Service ClusterIP port 80 |
| `k8s/ingress.yaml` | Ingress host `taskflow.localhost` |
| `k8s/secret.yaml` | Secret `taskflow-secret` |

```bash
# Appliquer tous les manifests d'un coup
kubectl apply -f k8s/

# Vérifier les pods (attendre ~30s qu'ils passent en Running)
kubectl get pods -n taskflow

# Vue globale de toutes les ressources du namespace
kubectl get all -n taskflow

# Vérifier l'Ingress
kubectl get ingress -n taskflow
```


```bash
# (Si nécessaire) Ajouter taskflow.localhost dans /etc/hosts
echo "127.0.0.1 taskflow.localhost" | sudo tee -a /etc/hosts
# Sur Windows (PowerShell en admin) :
# Add-Content C:\Windows\System32\drivers\etc\hosts "127.0.0.1 taskflow.localhost"

# Tester l'accès via l'Ingress
curl http://taskflow.localhost:8080
```


```bash
# Voir les logs d'un pod
kubectl logs -n taskflow <nom-du-pod>

# Décrire un pod (utile en cas d'erreur)
kubectl describe pod -n taskflow <nom-du-pod>

# Lister les clusters k3d
k3d cluster list

# Vérifier que le port 8080 est bien mappé
docker ps | grep k3d

# Supprimer le cluster en fin de TP
k3d cluster delete taskflow
```
