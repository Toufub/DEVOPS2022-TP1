# TP 3 - ANSIBLE

## Connexion SSH

Pour se connecter en SSH au Centos, tout d'abord on créé le fichier key, on insère la clef privée à l’intérieur et on change ses droits d'accès. Suite à quoi on peut tester la connection SSH.
```
touch key
sudo chmod 400 key
ssh -i ./ssh_key centos@eloi.cambray-lagassy.takima.cloud
```

## Test de la connexion avec ANSIBLE

Pour tester la connexion on va ajouter  la machine dans le fichier /etc/ansible/hosts qui correspond à l'inventaire :
```
sudo nano /etc/ansible/hosts
``` 
Voici donc le contenu du fichier :
```
# This is the default ansible 'hosts' file.
#
# It should live in /etc/ansible/hosts
#
#   - Comments begin with the '#' character
#   - Blank lines are ignored
#   - Groups of hosts are delimited by [header] elements
#   - You can enter hostnames or ip addresses
#   - A hostname/ip can be a member of multiple groups

centos@eloi.cambray-lagassy.takima.cloud
```

On va donc essayer de ping :
```
ansible all -m ping
```
Cela ne fonctionne pas car on doit lui fournir la clef SSH :
```
ansible all -m ping --private-key=ssh_key -u centos
centos@eloi.cambray-lagassy.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```
## Intro
### Inventories

Tout d'abord après avoir créé une architecture de dossier spécifique au projet Ansible, on créé le fichier setup.yml dont voici le contenu :
```
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ./ssh_key
  children:
    prod:
      hosts: centos@eloi.cambray-lagassy.takima.cloud
``` 

Et on lance le ping de ce setup en fournissant le fichier de setup en paramètre :
```
sudo ansible all -i inventories/setup.yml -m ping
```

### Facts

Grace au module de setup d'ansible, il est possible de récupérer les infos d'OS de la machine :
```
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
centos@eloi.cambray-lagassy.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "8",
        "ansible_distribution_release": "NA",
        "ansible_distribution_version": "8",
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false
}
```

Pour supprimer l'Apache du TD, on utilise :
```
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```
## Playbooks

On créé le fichier playbook.yml dans le dossier touch avec ce contenu :
```
 hosts: all
  gather_facts: false
  become: yes
  tasks:
    - name: Test connection
      ping:
```
Ce playbook va simplement exécuter la commande ping pour tester la connexion sur toutes les machines de l'inventaire.
Pour le lancer, on utilise :
```
ansible-playbook -i inventories/setup.yml playbook.yml
```

Pour continuer, on va installer Docker sur la machine via ce playbook :
```
  tasks:
    - name: Clean packages
      command:
        cmd: dnf clean -y packages
    - name: Install device-mapper-persistent-data
      dnf:
        name: device-mapper-persistent-data
        state: latest
    - name: Install lvm2
      dnf:
        name: lvm2
        state: latest
    - name: add repo docker
      command:
        cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    - name: Install Docker
      dnf:
        name: docker-ce
        state: present
    - name: install python3
      dnf:
        name: python3
    - name: Pip install
      pip:
        name: docker
    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

Pour vérifier, on se connecte simplement en SSH sur la machine et on lance *docker --version*

### Using role

Pour créer un rôle on utilise :
```
ansible-galaxy init roles/docker
```
Cela créé une architecture de dossier qui seront utilisé par le playbook via ces lignes :
```
- hosts: all
  roles:
    - docker
```

## Déploiement de l'application

Pour mettre en place l'application j'ai tout d'abord créé les différents rôles chargés de mettre en place les différentes fonctionnalités :
```
ansible-galaxy init roles/docker # installe docker
ansible-galaxy init roles/network # créé le réseau docker
ansible-galaxy init roles/database # met en place le container database
ansible-galaxy init roles/app # met en place le backend java
ansible-galaxy init roles/proxy # met en place le proxy
```

Voici le contenu des fichiers tasks respectifs :

### Network
```
- name: Create the network
  docker_network:
    name: app_network
```
### Database
```
- name: Run Database
  docker_container:
    name: postgres
    image: toufub/postgres:1.0
    ports:
      - "5432:5432"
    network_mode: app_network
```
### App
```
- name: Run App
  docker_container:
    name: backend-api
    image: toufub/backend-api:1.0
    ports:
      - "8080:8080"
    network_mode: app_network
```
### Proxy
```
- name: Run Proxy
  docker_container:
    name: proxy
    image: toufub/http_basic:1.0
    ports:
      - "80:80"
    network_mode: app_network
```

### Fin
Pour vérifier le fonctionnement, on se rend simplement sur http://eloi.cambray-lagassy.takima.cloud/students/1 et on constate le fonctionnement.
