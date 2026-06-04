```bash
node --version
npm --version
ansible web -m ping
```

Résultat attendu

```bash
v18.20.4

10.8.2

taskflow-web1 | SUCCESS => {
    "ping": "pong"
}

taskflow-web2 | SUCCESS => {
    "ping": "pong"
}
```

```bash
nano ansible/roles/taskflow/templates/nginx-taskflow.conf.j2
```

Contenu

```jinja2
# {{ ansible_managed }}
# Configuration nginx pour {{ app_name }} — générée par Ansible

server {
    listen {{ app_http_port }};
    server_name {{ app_server_name }};

    root {{ app_web_root }};
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log  /var/log/nginx/{{ app_name }}_error.log;
}
```


```bash
nano ansible/roles/taskflow/handlers/main.yml
```

Contenu

```yaml

- name: Reload nginx
  ansible.builtin.service:
    name: nginx
    state: reloaded
```


```bash
nano ansible/roles/taskflow/tasks/main.yml
```

Contenu

```yaml

- name: Installer nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Créer la racine web de l'application
  ansible.builtin.file:
    path: "{{ app_web_root }}"
    state: directory
    owner: www-data
    group: www-data
    mode: "0755"

- name: Copier les fichiers buildés de l'application
  ansible.posix.synchronize:
    src: "{{ app_dist_local }}/"
    dest: "{{ app_web_root }}/"
    delete: true
    recursive: true
  notify: Reload nginx

- name: Déployer la configuration nginx de TaskFlow
  ansible.builtin.template:
    src: nginx-taskflow.conf.j2
    dest: "/etc/nginx/sites-available/{{ app_name }}.conf"
    owner: root
    group: root
    mode: "0644"
  notify: Reload nginx

- name: Activer le site TaskFlow
  ansible.builtin.file:
    src: "/etc/nginx/sites-available/{{ app_name }}.conf"
    dest: "/etc/nginx/sites-enabled/{{ app_name }}.conf"
    state: link
  notify: Reload nginx

- name: Désactiver le site nginx par défaut
  ansible.builtin.file:
    path: /etc/nginx/sites-enabled/default
    state: absent
  notify: Reload nginx

- name: S'assurer que nginx est démarré et activé
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true
```


```bash
nano ansible/playbooks/deploy-app.yml
```
Contenu

```yaml

- name: Construire l'application TaskFlow localement
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Installer les dépendances npm
      community.general.npm:
        path: "{{ playbook_dir }}/.."
        ci: true

    - name: Builder l'application
      ansible.builtin.command:
        cmd: npm run build
        chdir: "{{ playbook_dir }}/.."
        creates: "{{ playbook_dir }}/../dist/index.html"

- name: Déployer TaskFlow sur les serveurs web
  hosts: web
  become: true
  gather_facts: true

  roles:
    - role: taskflow
```


Exécution du déploiement

```bash
cd ansible
ansible-playbook playbooks/deploy-app.yml
```

Résultat

```bash
PLAY RECAP

localhost     : ok=2 changed=2 unreachable=0 failed=0
taskflow-web1 : ok=8 changed=6 unreachable=0 failed=0
taskflow-web2 : ok=8 changed=6 unreachable=0 failed=0
```

```bash
curl http://192.168.64.11
curl http://192.168.64.12
```

Résultat

```html
<title>TaskFlow</title>
```

```bash
ansible-playbook playbooks/deploy-app.yml
```


```bash
localhost     : ok=2 changed=0 unreachable=0 failed=0
taskflow-web1 : ok=8 changed=0 unreachable=0 failed=0
taskflow-web2 : ok=8 changed=0 unreachable=0 failed=0
```
