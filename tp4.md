```bash
# Vérifier le cluster
k3d cluster list
kubectl get nodes

# Installer la dépendance Python (si ce n'est pas déjà fait)
pip install kubernetes
# ou : pip3 install kubernetes

# Vérifier l'installation
python3 -c "import kubernetes; print(kubernetes.__version__)"
```

```bash
# Installer la collection kubernetes.core (déjà déclarée dans requirements.yml)
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list kubernetes.core
```

> Fichiers créés dans `ansible/playbooks/` :

| Fichier | Rôle |
|---|---|
| `vars/k8s.yml` | Variables du déploiement (namespace, image, replicas, resources, config, secret) |
| `templates/k8s/deployment.yaml.j2` | Template Jinja2 du Deployment |
| `k8s-deploy.yml` | Playbook (namespace, configmap, secret, deployment, service, ingress, wait) |
| `vars/secrets.yml` | Secret en clair (chiffrement Vault au J5) |

```bash
# Premier déploiement
cd ansible
ansible-playbook playbooks/k8s-deploy.yml

# Vérifier les pods (2 attendus, Running)
kubectl get pods -n taskflow

# Vérifier l'accès
curl http://taskflow.localhost:8080

# Prouver l'idempotence (attendu : changed=0)
ansible-playbook playbooks/k8s-deploy.yml
```

```bash
# Scaling déclaratif : modifier taskflow_replicas dans ansible/playbooks/vars/k8s.yml
# taskflow_replicas: 3

ansible-playbook playbooks/k8s-deploy.yml
kubectl get pods -n taskflow

# Revenir à 2 replicas dans k8s.yml puis relancer
ansible-playbook playbooks/k8s-deploy.yml
kubectl get pods -n taskflow
```

```bash
# Dépannage
# ModuleNotFoundError: No module named 'kubernetes' -> pip install kubernetes
# Unable to connect to the server -> vérifier k3d cluster list / kubectl get nodes
# Timeout sur "Attendre que le déploiement soit disponible" -> réimporter l'image
k3d image import taskflow:1.0.0 -c taskflow
```
