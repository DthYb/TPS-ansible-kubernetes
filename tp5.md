```bash
# Installer ansible-lint si nécessaire
pip install ansible-lint

# Vérifier
ansible-lint --version
```

```bash
# Chiffrer secrets.yml avec Ansible Vault
cd ansible
ansible-vault encrypt playbooks/vars/secrets.yml
# New Vault password: taskflow-lab-2024
# Confirm New Vault password: taskflow-lab-2024

# Vérifier que le fichier est chiffré
cat playbooks/vars/secrets.yml

# Voir le contenu en clair
ansible-vault view playbooks/vars/secrets.yml

# Modifier une valeur chiffrée (ouvre l'éditeur, rechiffre à la sauvegarde)
ansible-vault edit playbooks/vars/secrets.yml
```

> Vérifier que `ansible/playbooks/templates/k8s/deployment.yaml.j2` contient bien `resources`, `livenessProbe` et `readinessProbe` (créés au J4), et que `playbooks/vars/k8s.yml` définit `taskflow_resources`.

```bash
# Lancer ansible-lint sur les playbooks du projet
ansible-lint playbooks/provision.yml playbooks/deploy-app.yml playbooks/k8s-deploy.yml
echo "Exit code: $?"
```

> Erreurs bloquantes courantes : `yaml[truthy]` (remplacer `yes`/`no` par `true`/`false`), `fqcn[action-core]` (utiliser le nom complet du module), `no-changed-when` (ajouter `changed_when` sur les tâches `command`/`shell`), `risky-file-permissions` (ajouter `mode` sur `file`/`copy`).

```bash
# Exemple de correction no-changed-when dans deploy-app.yml
# - name: Builder l'application (vite build -> dist/)
#   ansible.builtin.command:
#     cmd: npm run build
#     chdir: "{{ playbook_dir }}/.."
#     creates: "{{ playbook_dir }}/../dist/index.html"
#   changed_when: false
```

```bash
# (Optionnel) Repartir d'un cluster propre
k3d cluster delete taskflow
k3d cluster create taskflow --port "8080:80@loadbalancer"
k3d image import taskflow:1.0.0 -c taskflow

# Provisioning (idempotent, changed=0 attendu)
ansible-playbook playbooks/provision.yml

# Déploiement de l'application sur les VMs
ansible-playbook playbooks/deploy-app.yml

# Déploiement Kubernetes complet avec Vault
ansible-playbook playbooks/k8s-deploy.yml --ask-vault-pass
```

```bash
# Vérifier les probes sur les pods déployés
kubectl describe pod -n taskflow -l app=taskflow | grep -A 10 "Liveness\|Readiness"

# État des pods
kubectl get pods -n taskflow

# Accès à l'application
curl http://taskflow.localhost:8080

# Vérifier le Secret (base64, jamais en clair)
kubectl get secret taskflow-secret -n taskflow -o yaml
```

```bash
# Dépannage
# Decryption failed -> mauvais mot de passe vault ou --ask-vault-pass manquant
ansible-vault view playbooks/vars/secrets.yml

# ansible-lint qui scanne aussi les collections installées -> limiter le scope
ansible-lint playbooks/ roles/

# CrashLoopBackOff -> inspecter les logs, augmenter initialDelaySeconds si besoin
kubectl logs -n taskflow -l app=taskflow

# Unable to retrieve file contents -> vérifier le chemin du template
ls playbooks/templates/k8s/deployment.yaml.j2
```
