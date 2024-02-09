# Compte rendu TP3 Ansible + TP Extra

## Introduction

On setup notre server ansible, on spécifie l'emplacement de notre clé ssh et notre nom de domaine.

**setup.yml** 

```
all:  
  vars:  
    ansible_user: centos  
    ansible_ssh_private_key_file: /mnt/c/Grégory/CPE/4IRC/DevOps/DevOpsTPs/TP3/id_rsa  
  children:  
    prod:  
      hosts: centos@gregory.spinar.takima.cloud
```

**remove httpd**

```
ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become

```

On retire le server Apache httpd qu'on avait installé manuellement sur Ansible. Avec ansible, on décrie simplement l'état de notre serveur et on laisse ansible mettre à jour automatiquement.

### Playbook 

```
- hosts: all  
  gather_facts: false  
  become: true  
  roles:  
    - docker  
    - network  
    - database  
    - app  
    - front  
    - proxy  
  vars_files:   
    - secrets.yml
```

On décrit ici les différents rôles qui installeront les images sur notre server ansible mais aussi le fichier secret.yml qui contient nos variables d'environnements encryptées.
## Roles

```
---  
# tasks file for roles/docker  
# Install Docker  
  
- name: Install device-mapper-persistent-data  
  yum:  
    name: device-mapper-persistent-data  
    state: latest  
  
- name: Install lvm2  
  yum:  
    name: lvm2  
    state: latest  
  
- name: add repo docker  
  command:  
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo  
  
- name: Install Docker  
  yum:  
    name: docker-ce  
    state: present  
  
- name: Install python3  
  yum:  
    name: python3  
    state: present  
  
- name: Install docker with Python 3  
  pip:  
    name: docker  
    executable: pip3  
  vars:  
    ansible_python_interpreter: /usr/bin/python3  
  
- name: Make sure Docker is running  
  service: name=docker state=started  
  tags: docker
```

Le playbook appelle le rôle docker qui installe Docker et Python3 sur notre server Ansible. On va maintenant écrire les rôles pour : Base de données, Backend, Proxy, Network, Front. 

Exemple task Backend : 

```
---  
# tasks file for roles/app  
- name: Run App  
  community.docker.docker_container:  
    name: backend  
    pull: true  
    recreate: true  
    image: gregoryspn/tp1-backend:latest  
    networks:  
      - name: app-network  
    env:   
      DB_URL: "{{db_url}}"  
      DB_USER: "{{db_user}}"  
      DB_PASSWORD: "{{db_password}}"  
  
- name: Run App 2  
  community.docker.docker_container:  
    name: backend2  
    pull: true  
    recreate: true  
    image: gregoryspn/tp1-backend:latest  
    networks:  
      - name: app-network  
    env:  
      DB_URL: "{{db_url}}"  
      DB_USER: "{{db_user}}"  
      DB_PASSWORD: "{{db_password}}"
```
## Front

On modifie la configuration de notre proxy pour qu'il fasse la redirection entre notre API et le Front. 

```
ProxyPass / http://front:80/  
ProxyPassReverse / http://front:80/
```

## Continuous Deployment 

On utilise un workflow afin de déployer automatiquement notre application sur le serveur Ansible lorsque nous mettons à jour la branche main de notre Git.

**deploy.yml**

```
name: CI devops deploy 2024  
on:  
  workflow_run:  
    workflows:  
      - "CI devops docker 2024"  
    types:  
        - completed  
    branches:  
        - main  
  
jobs:  
  deploy_front:   
    if: ${{github.event.workflow_run.conclusion == 'success'}}  
    runs-on: ubuntu-22.04  
    steps:  
      - name: Checkout code  
        uses: actions/checkout@v2.5.0  
  
      - name: Run playbook  
        uses: dawidd6/action-ansible-playbook@v2  
        with:  
          playbook: playbook.yml  
          directory: ansible  
          key: ${{secrets.SSH_KEY}}  
          inventory: |  
            [all]  
            ${{ secrets.DEPLOY_HOST }}  
          vault_password: ${{ secrets.VAULT_PASSWORD }}  
          options: |  
            -u centos
```

Ce fichier exécute en fait cette commande : 

```
ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-password
```

Il lance le déploiement uniquement lorsque le workflow "docker" (chargé de lancer nos images) s'est bien déroulé. Les variable ssh_key, deploy_host, vault_password sont stockées dans le fichier Vault secret.yml. 

# TP Extra

Notre but est maintenant de lancer 2 backend sur notre proxy.

On modifie dans un premier temps le rôle de notre backend en rajoutant un 2ème backend, tous deux utilisant la même image tp1-backend.

**main.yml**

```
# tasks file for roles/app  
- name: Run App  
  community.docker.docker_container:  
    name: backend  
    pull: true  
    recreate: true  
    image: gregoryspn/tp1-backend:latest  
    networks:  
      - name: app-network  
    env:   
      DB_URL: "{{db_url}}"  
      DB_USER: "{{db_user}}"  
      DB_PASSWORD: "{{db_password}}"  
  
- name: Run App 2  
  community.docker.docker_container:  
    name: backend2  
    pull: true  
    recreate: true  
    image: gregoryspn/tp1-backend:latest  
    networks:  
      - name: app-network  
    env:  
      DB_URL: "{{db_url}}"  
      DB_USER: "{{db_user}}"  
      DB_PASSWORD: "{{db_password}}"
```

Puis on configure notre proxy avec un cluster qui associera au port 80 nos 2 backend : 

**httpd.conf**

```
<VirtualHost *:80>  
ProxyPreserveHost On  
<Proxy "balancer://mycluster">  
     BalancerMember "http://backend:8080"  
     BalancerMember "http://backend2:8080"  
</Proxy>  
ProxyPass /api/ "balancer://mycluster/"  
ProxyPassReverse /api/ "balancer://mycluster/"  
ProxyPass / http://front:80/  
ProxyPassReverse / http://front:80/  
</VirtualHost>  
LoadModule proxy_module modules/mod_proxy.so  
LoadModule proxy_http_module modules/mod_proxy_http.so  
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so  
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so  
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
```



