# KVM Ansible Lab

### 4.1. Topologie de lab

Pour reproduire cette topologie avec libvirt/KVM, vous avez besoin d'un hôte de virtualisation physique (ou virtuel) avec les instructions de virtualisation activées et une installation Centos8 (ou Ubuntu) à jour.

Après avoir installé les pré-requis, nous allons créer trois machines virtuelles : controller, node1 et node2 et approvisionner la solution pour assurer une gestion des deux noeuds par la machine de contrôle.

Nous conseillons vivement d'utiliser des instances dans le nuage pour des serveurs "bare-metal"[^3] chez [Scaleway](https://www.scaleway.com/fr/serveurs-bare-metal/), [Packet](https://metal.equinix.com/product/servers/), [Hetzner](https://www.hetzner.com/dedicated-rootserver) ou encore [OVH](https://www.ovhcloud.com/fr/bare-metal/).

[^3]: Un "serveur bare-metal" est un serveur informatique qui est un "serveur physique à locataire unique". Ce terme est utilisé de nos jours pour le distinguer des formes modernes de virtualisation et d'hébergement en nuage. ([source](https://en.wikipedia.org/wiki/Bare-metal_server))

Dans un premier temps, il s'agira de configurer l'hôte de virtualisation. Ensuite, on créera les trois machines virtuelles. Enfin, nous configurerons le "controller" en y installant Ansible, en y plaçant un fichier d'inventaire et un fichier de configuration par défaut. Une paire de clés SSH sera générée et permettra d'authentifier le controller auprès des noeuds.

Si vous avez un bon serveur de virtualisation sous la main installé en Centos 8, la configuration "se déroule" en quelques minutes.

Cette topologie est constituée de quatre machines virtuelles installée avec [l'image Cloud de Centos 8](https://cloud.centos.org/centos/8/x86_64/images/CentOS-8-GenericCloud-8.2.2004-20200611.2.x86_64.qcow2) et des utilisateurs "root" et "ansible" configurés avec [un mot de passe trivial](https://github.com/goffinet/kvm-ansible-lab/blob/7d70bfa8e170b54b50d0f5a26e7747e1fe1ed47c/inventory/lab.yml#L8).

### 4.2. Créer la topologie avec Ansible

S'il fallait créer manuellement cette topologie, il serait nécessaire de procéder par étapes :

1. Installation de quelques pré-requis (Ansible, KVM, Libvirt)
2. Téléchargement des images
3. Création de quatre VM : "controller", "node0", "node1" et "node2"

Heureusement un livre de jeu a été préparé pour faciliter ce déploiement.

Sur l'hôte de virtualisation, nous allons "cloner" avec Git le livre de jeux [kvm-ansible-lab](https://github.com/goffinet/kvm-ansible-lab.git) dans le dossier `~/kvm-ansible-lab` et exécuter celui-ci, il est conseillé d'être 'root' sur le serveur de virtualisation. Cela tient en quelques lignes.

```bash
yum -y install git || apt -y install git
git clone --recursive https://github.com/goffinet/kvm-ansible-lab.git \
~/kvm-ansible-lab
cd ~/kvm-ansible-lab
./run.sh
```

Le script `run.sh` vérifie que Ansible soit installé et vous propose le cas échéant d'y rémédier (en Bash). Ensuite, il lance un livre de jeu Ansible qui construit la topologie.

### 4.3. Prendre connaissance de l'infrastructure créée

Entretemps, vous pouvez vous informer sur ce que vous exécutez en consultant le [projet kvm-ansible-lab](https://github.com/goffinet/kvm-ansible-lab/) :

- le script [run.sh](https://github.com/goffinet/kvm-ansible-lab/blob/master/run.sh),
- le livre de jeu [virt-infra.yml](https://github.com/goffinet/kvm-ansible-lab/blob/master/virt-infra.yml) que le script lance,
- et l'inventaire de la topologie [inventory/lab.yml](https://github.com/goffinet/kvm-ansible-lab/blob/master/inventory/lab.yml) en format YAML.

La documentation nous indique que l'on peut modifier l'état de la topologie en manipulant une variable "`virt_infra_state`". Ici pour détruire les machines virtuelles et les retirer de la gestion de libvirt, on valorise cette variable `"virt_infra_state=undefined"` sur l'exécution du livre de jeu :

```bash
./run.sh -e "virt_infra_state=undefined"
```

ou directement avec le livre de jeu :

```bash
ansible-playbook virt-infra.yml -e "virt_infra_state=undefined"
```

### 4.4. Gestion Ansible

Dès que le lab sera monté, les machines virtuelles seront directement gérables à partir de l'hôte de virtualisation :

```
ansible -m ping all
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node0 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
controller | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
node1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Aussi, vous pourrez éventuellement vous connecter sur le contrôleur :

```bash
ssh root@controller
```

et créer un projet de base :


```bash
dnf -y install epel-release && dnf -y install ansible
PROJECT="lab"
mkdir ~/${PROJECT}
cd ~/${PROJECT}
cat <<  EOF > ~/${PROJECT}/ansible.cfg
[defaults]
inventory = ./inventory
host_key_checking = False
EOF
cat <<  EOF > ~/${PROJECT}/inventory
[nodes]
node0
node1
node2

[all:vars]
ansible_connection=ssh
ansible_user=ansible
ansible_ssh_pass=testtest
ansible_become=yes
ansible_become_user=root
ansible_become_method=sudo
EOF
ansible -m ping all
```
