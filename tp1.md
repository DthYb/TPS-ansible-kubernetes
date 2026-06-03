sudo apt update
sudo apt install -y ansible openssh-client

puis 
multipass launch --name taskflow-web1 --cpus 1 --memory 512M --disk 5G
multipass launch --name taskflow-web2 --cpus 1 --memory 512M --disk 5G

multipass list
Name                    State             IPv4             Image
taskflow-web1           Running           172.30.131.189   Ubuntu 26.04 LTS
taskflow-web2           Running           172.30.129.90    Ubuntu 26.04 LTS

Dans wsl :
mkdir -p ~/.ssh
ssh-keygen -t ed25519 -f ~/.ssh/taskflow_lab -N "" -C "taskflow-lab"
mdp : key
Cmdp: key
multipass transfer $env:USERPROFILE\.ssh\taskflow_lab.pub taskflow-web1:/tmp/taskflow_lab.pub

Sinon dans powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE/.ssh/taskflow_lab -C "taskflow-lab"
ou
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\taskflow_lab -N "" -C "taskflow-lab"

multipass transfer $env:USERPROFILE\.ssh\taskflow_lab.pub taskflow-web1:/tmp/taskflow_lab.pub
multipass exec taskflow-web1 -- bash -c "mkdir -p ~/.ssh && cat /tmp/taskflow_lab.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

multipass transfer $env:USERPROFILE\.ssh\taskflow_lab.pub taskflow-web2:/tmp/taskflow_lab.pub
multipass exec taskflow-web2 -- bash -c "mkdir -p ~/.ssh && cat /tmp/taskflow_lab.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

pour tester :

ssh -i $env:USERPROFILE\.ssh\taskflow_lab ubuntu@172.30.131.189 "hostname"
ssh -i $env:USERPROFILE\.ssh\taskflow_lab ubuntu@172.30.129.90 "hostname"

Resultat :

taskflow-web1
taskflow-web2

Retourne WSl

ansible --version

/mnt/c/Users/osiri/.ssh/taskflow_lab

j'ai eu un problème de lisibilité de l'ip

multipass shell taskflow-web1

sudo apt update
sudo apt install -y ansible python3-pip
ansible --version

ssh-keygen -t ed25519 -f ~/.ssh/taskflow_lab -N "" -C "taskflow-lab"

cat ~/.ssh/taskflow_lab.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

ssh-copy-id -i ~/.ssh/taskflow_lab.pub ubuntu@172.30.129.90

ssh -i ~/.ssh/taskflow_lab ubuntu@172.30.131.189 "hostname"
ssh -i ~/.ssh/taskflow_lab ubuntu@172.30.129.90 "hostname"

Dans WSL

mkdir -p ~/taskflow-lab/ansible/inventory/group_vars
mkdir -p ~/taskflow-lab/ansible/playbooks
cd ~/taskflow-lab

nano ansible/ansible.cfg


[defaults]
inventory = inventory/hosts.ini
host_key_checking = False
roles_path = roles
retry_files_enabled = False
stdout_callback = yaml
interpreter_python = auto_silent

[privilege_escalation]
become = False


nano ansible/inventory/hosts.ini


[web]
taskflow-web1 ansible_host=172.30.131.189
taskflow-web2 ansible_host=172.30.129.90

[web:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/taskflow_lab

[k8s_control]
localhost ansible_connection=local

nano ansible/inventory/group_vars/all.yml


---
deploy_user: deploy
timezone: Europe/Paris

base_packages:
  - curl
  - git
  - vim
  - ufw
  - htop


  nano ansible/inventory/group_vars/web.yml

  ---
app_name: taskflow
app_web_root: /var/www/taskflow
app_server_name: taskflow.local
app_http_port: 80

app_dist_local: "{{ playbook_dir }}/../dist"


nano ansible/playbooks/provision.yml

---
- name: Provisionner les serveurs web TaskFlow
  hosts: web
  become: true
  gather_facts: true

  tasks:
    - name: Mettre a jour le cache APT
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: Installer les paquets de base
      ansible.builtin.apt:
        name: "{{ base_packages }}"
        state: present

    - name: Definir le fuseau horaire
      community.general.timezone:
        name: "{{ timezone }}"

    - name: Creer l'utilisateur de deploiement
      ansible.builtin.user:
        name: "{{ deploy_user }}"
        shell: /bin/bash
        groups: sudo
        append: true
        state: present

    - name: Autoriser SSH dans le pare-feu
      community.general.ufw:
        rule: allow
        name: OpenSSH

    - name: Autoriser le trafic HTTP
      community.general.ufw:
        rule: allow
        port: "80"
        proto: tcp

    - name: Activer UFW
      community.general.ufw:
        state: enabled
        policy: deny
        direction: incoming

  post_tasks:
    - name: Afficher un recapitulatif
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} provisionne ({{ ansible_distribution }} {{ ansible_distribution_version }})"

cd ~/taskflow-lab/ansible
ansible web -m ping


ansible-playbook playbooks/provision.yml

taskflow-web1 : ok=... changed=0 unreachable=0 failed=0
taskflow-web2 : ok=... changed=0 unreachable=0 failed=0

ansible web -m command -a "id deploy"

ansible web -m command -a "ufw status" --become

VM taskflow-web1 Running : OK
VM taskflow-web2 Running : OK
SSH vers les deux VMs : OK
Inventaire Ansible avec les bonnes IP : OK
ansible web -m ping : SUCCESS
Premier provision.yml : failed=0
Deuxième provision.yml : changed=0
Utilisateur deploy existe : OK
UFW actif : OK


cd ~/taskflow-lab/ansible
ansible web -m ping
ansible-playbook playbooks/provision.yml
ansible-playbook playbooks/provision.yml
ansible web -m command -a "id deploy"
ansible web -m command -a "ufw status" --become