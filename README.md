# TP 3

## 3-1 Inventaire et commandes de base

### Inventaire

`ansible/inventories/setup.yml`

```yaml
all:
  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: /home/nicolas/.ssh/id_rsa_takima
  children:
    prod:
      hosts: nicolas.estermann.takima.school
```

L'inventaire indique à Ansible sur quelles machines agir et comment s'y connecter. Le groupe racine `all` définit des variables communes à tous les hôtes : l'utilisateur SSH (`admin`, utilisateur par défaut sur Debian) et le chemin de la clé privée. Plus besoin de passer `-u` et `--private-key` à chaque commande. Le sous-groupe `prod` contient le serveur. On pourrait ajouter d'autres groupes (database, front, staging...) pour cibler des machines différentes.

### Commandes de base

```bash
ansible all -i inventories/setup.yml -m ping
```

Vérifie qu'Ansible peut se connecter en SSH, s'authentifier et exécuter Python sur le serveur. Réponse attendue : `pong`.

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le module `setup` récupère les *facts*, des variables découvertes automatiquement sur l'hôte et préfixées par `ansible_`. Le filtre ne garde que les informations sur l'OS (Debian et sa version).

```bash
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

Le module `apt` décrit l'état voulu d'un paquet : `state=absent` signifie qu'Apache ne doit pas être installé. `--become` exécute la commande en root, nécessaire pour gérer les paquets. Au premier lancement, la commande renvoie `changed: true` (Apache est supprimé). Au second, `changed: false` (l'état voulu est déjà atteint). C'est l'**idempotence** : on décrit un état cible, pas une suite d'actions.

## 3-2 Playbook

### Structure

```
ansible/
├── inventories/
│   └── setup.yml
├── playbook.yml
└── roles/
    └── docker/
        ├── handlers/
        │   └── main.yml
        └── tasks/
            └── main.yml
```

### `playbook.yml`

```yaml
- hosts: all
  gather_facts: true
  become: true
  roles:
    - docker
```

- `hosts: all` : le playbook s'applique à tous les hôtes de l'inventaire.
- `gather_facts: true` : récupère les facts, nécessaires pour `ansible_facts['distribution_release']` (nom de code de la version Debian) utilisé dans l'URL du dépôt Docker.
- `become: true` : exécute les tâches en root (installation de paquets, gestion de services).
- `roles` : appelle le rôle `docker`, ce qui garde le playbook court et rend l'installation réutilisable.

### Rôle `docker`

Le rôle est créé avec `ansible-galaxy init roles/docker`. Seuls les dossiers `tasks` (liste des tâches du rôle) et `handlers` (actions déclenchées sur notification) sont conservés.

`roles/docker/tasks/main.yml`

```yaml
- name: Install required packages
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
      - python3-venv
    state: latest
    update_cache: yes

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/debian/gpg
    state: present

- name: Add Docker APT repository
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/debian {{ ansible_facts['distribution_release'] }} stable"
    state: present
    update_cache: yes

- name: Install Docker
  apt:
    name: docker-ce
    state: present

- name: Install Python3 and pip3
  apt:
    name:
      - python3
      - python3-pip
    state: present

- name: Create a virtual environment for Docker SDK
  command: python3 -m venv /opt/docker_venv
  args:
    creates: /opt/docker_venv

- name: Install Docker SDK for Python in virtual environment
  pip:
    name: docker
    virtualenv: /opt/docker_venv

- name: Make sure Docker is running
  service:
    name: docker
    state: started
  tags: docker
```

| Tâche | Rôle |
|---|---|
| Install required packages | Installe les prérequis (HTTPS pour apt, certificats, curl, gnupg, venv) et met à jour le cache apt |
| Add Docker GPG key | Ajoute la clé de signature officielle de Docker pour vérifier les paquets |
| Add Docker APT repository | Ajoute le dépôt officiel Docker correspondant à la version de Debian |
| Install Docker | Installe `docker-ce` |
| Install Python3 and pip3 | Garantit la présence de Python et pip |
| Create a virtual environment | Crée un venv dans `/opt/docker_venv`. `creates` rend la tâche idempotente : elle n'est exécutée que si le dossier n'existe pas |
| Install Docker SDK | Installe le SDK Python `docker` dans le venv, requis par les modules `community.docker` (docker_container, docker_network). Le module `pip` est utilisé à la place de `command` car il vérifie si le paquet est déjà installé : la tâche est idempotente |
| Make sure Docker is running | Vérifie que le service Docker est démarré |

### Exécution

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml
```

Vérification :

```bash
ansible all -i inventories/setup.yml -m command -a "docker --version" --become
```

Relancé une seconde fois, le playbook renvoie `changed=0` : le rôle est entièrement idempotent.
