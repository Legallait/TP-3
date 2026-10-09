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

## 3-3 Déploiement de l'application avec docker_container

### Structure

```
ansible/
├── inventories/
│   └── setup.yml
├── playbook.yml
└── roles/
    ├── docker/
    ├── network/
    ├── database/
    ├── app/
    └── proxy/
```

Chaque partie de l'application a son propre rôle, ce qui permet de les faire évoluer ou de les réutiliser séparément.

### `playbook.yml`

```yaml
- hosts: all
  gather_facts: true
  become: true
  roles:
    - docker

- hosts: all
  gather_facts: false
  become: true
  vars:
    ansible_python_interpreter: /opt/docker_venv/bin/python
    dockerhub_user: nicolases
    db_image: tp-devops-database
    api_image: tp-devops-simple-api
    httpd_image: tp-devops-httpd
    db_container: database
    api_container: simple-api
    db_name: db
    db_user: usr
    db_password: pwd
  roles:
    - network
    - database
    - app
    - proxy
```

Le playbook est découpé en deux plays :

- le premier installe Docker avec le Python système, nécessaire au module `apt` ;
- le second utilise `ansible_python_interpreter: /opt/docker_venv/bin/python`, car les modules `community.docker` ont besoin du SDK Python `docker`, installé dans le venv.

Toutes les valeurs (images, noms de conteneurs, identifiants) sont centralisées dans `vars` et réutilisées dans les rôles avec la syntaxe Jinja2 `{{ variable }}`.

### Rôle `network`

```yaml
- name: Create app network
  community.docker.docker_network:
    name: app-network
```

Crée un réseau Docker dédié. Les conteneurs connectés à ce réseau se joignent par leur nom (DNS interne de Docker) : l'API contacte `database`, le proxy contacte `simple-api`.

### Rôle `database`

```yaml
- name: Run database
  community.docker.docker_container:
    name: "{{ db_container }}"
    image: "{{ dockerhub_user }}/{{ db_image }}:latest"
    pull: true
    restart_policy: always
    networks:
      - name: app-network
    env:
      POSTGRES_DB: "{{ db_name }}"
      POSTGRES_USER: "{{ db_user }}"
      POSTGRES_PASSWORD: "{{ db_password }}"
    volumes:
      - db-data:/var/lib/postgresql/data
```

### Rôle `app`

```yaml
- name: Run backend API
  community.docker.docker_container:
    name: "{{ api_container }}"
    image: "{{ dockerhub_user }}/{{ api_image }}:latest"
    pull: true
    restart_policy: always
    networks:
      - name: app-network
    env:
      DATABASE_HOST: "{{ db_container }}"
      SPRING_DATASOURCE_URL: "jdbc:postgresql://{{ db_container }}:5432/{{ db_name }}"
      SPRING_DATASOURCE_USERNAME: "{{ db_user }}"
      SPRING_DATASOURCE_PASSWORD: "{{ db_password }}"
```

### Rôle `proxy`

```yaml
- name: Run httpd proxy
  community.docker.docker_container:
    name: httpd
    image: "{{ dockerhub_user }}/{{ httpd_image }}:latest"
    pull: true
    restart_policy: always
    networks:
      - name: app-network
    published_ports:
      - "80:80"
```

### Paramètres utilisés

| Paramètre | Rôle |
|---|---|
| `name` | Nom du conteneur, utilisé aussi comme nom d'hôte sur le réseau Docker. `simple-api` doit correspondre au `ProxyPass` du `httpd.conf`, `database` à l'hôte de l'URL JDBC |
| `image` | Image publiée sur DockerHub par la CI du TP 2 |
| `pull: true` | Télécharge toujours la dernière version de l'image, ce qui permet de redéployer après un nouveau push |
| `restart_policy: always` | Redémarre le conteneur après un crash ou un reboot du serveur |
| `networks` | Connecte le conteneur au réseau `app-network` |
| `env` | Variables d'environnement. Pour la base, elles initialisent PostgreSQL. Pour l'API, `DATABASE_HOST` et `SPRING_DATASOURCE_*` surchargent les valeurs de `application.yml` sans reconstruire l'image |
| `volumes` | Volume nommé `db-data` pour que les données survivent à la recréation du conteneur |
| `published_ports` | Expose le port 80 du serveur. Seul le proxy publie un port : la base et l'API ne sont accessibles qu'à travers le réseau interne, ce qui réduit la surface d'attaque |

### Vérification

```bash
ansible all -i inventories/setup.yml -m command -a "docker ps" --become
```

Les trois conteneurs `database`, `simple-api` et `httpd` sont `Up`, et seul `httpd` expose `0.0.0.0:80->80/tcp`.

`http://nicolas.estermann.takima.school/departments` renvoie :

```json
[{"id": 1,"name": "IRC"},{"id": 2,"name": "ETI"},{"id": 3,"name": "CGP"}]
```
