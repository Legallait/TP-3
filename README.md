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

L'inventaire indique à Ansible sur quelles machines agir et comment s'y connecter. Le groupe racine `all` définit des variables communes à tous les hôtes : l'utilisateur SSH (`admin`, utilisateur par défaut sur Debian) et le chemin de la clé privée. Plus besoin de passer `-u` et `--private-key` à chaque commande. Le sous-groupe `prod` contient le serveur.

### Commandes de base

```bash
ansible all -i inventories/setup.yml -m ping
```

Vérifie qu'Ansible peut se connecter en SSH, s'authentifier et exécuter Python sur le serveur. Réponse attendue : `pong`.

```bash
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
```

Le module `setup` récupère les *facts*, des variables découvertes automatiquement sur l'hôte et préfixées par `ansible_`. 

```bash
ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become
```

Le module `apt` décrit l'état voulu d'un paquet : `state=absent` signifie qu'Apache ne doit pas être installé. `

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

### Rôle `network`

```yaml
- name: Create app network
  community.docker.docker_network:
    name: app-network
```

Crée un réseau Docker dédié. Les conteneurs connectés à ce réseau se joignent par leur nom : l'API contacte `database`, le proxy contacte `simple-api`.

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

## Continuous Deployment

Le dossier `ansible/` est ajouté au repo du TP 2 et un job `deploy` est ajouté au workflow `build-and-push.yml`. Chaque push sur `main` enchaîne : tests, build et push des images sur DockerHub, puis déploiement Ansible sur le serveur.
Repo GitHub (CI/CD et déploiement) : [Legallait/TP2_git](https://github.com/Legallait/TP2_git)

```yaml
  deploy:
    needs: build-and-push
    runs-on: ubuntu-24.04
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Ansible
        run: pip install ansible

      - name: Write SSH key
        run: |
          printf '%s\n' "${{ secrets.SSH_PRIVATE_KEY }}" | tr -d '\r' > key
          chmod 600 key

      - name: Deploy with Ansible
        working-directory: ansible
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
        run: ansible-playbook -i inventories/setup.yml playbook.yml -e ansible_ssh_private_key_file=../key
```

| Élément | Rôle |
|---|---|
| `needs: build-and-push` | Le déploiement n'a lieu que si toutes les images de la matrice ont été construites et poussées avec succès |
| `if: github.ref == 'refs/heads/main'` | Seule la branche `main` est déployée en production, `develop` est testée et buildée mais pas déployée |
| `SSH_PRIVATE_KEY` | La clé privée est stockée dans les secrets GitHub, jamais dans le repo. Elle est transmise au workflow appelé grâce à `secrets: inherit` dans `main.yml` |
| `tr -d '\r'` et `printf '%s\n'` | Suppriment les fins de ligne Windows et garantissent le retour à la ligne final, sans quoi OpenSSH refuse la clé (`error in libcrypto`) |
| `-e ansible_ssh_private_key_file=../key` | Les extra vars ont la priorité la plus haute et remplacent le chemin local défini dans l'inventaire |
| `ANSIBLE_HOST_KEY_CHECKING: "False"` | Évite la question interactive de confirmation de l'empreinte du serveur, qui bloquerait la CI |

Les conteneurs utilisant `pull: true`, chaque déploiement récupère les images `latest` qui viennent d'être poussées.

### Est-il sûr de déployer automatiquement chaque nouvelle image ?

Non. Une image cassée ou contenant une vulnérabilité partirait directement en production. 
Si le compte DockerHub ou un secret est compromis, une image malveillante serait déployée sans contrôle. 

## Front

Le front ([takima-training/devops-front](https://github.com/takima-training/devops-front)) est ajouté dans le dossier `front/` du repo, puis buildé, poussé et déployé comme les autres services.

### Routage

Le front possède sa propre route `/departments`, qui entre en conflit avec celle de l'API. L'API est donc déplacée sous `/api/` et le front occupe `/`. httpd reste l'unique point d'entrée et joue le rôle de reverse proxy :

```
Navigateur ──:80──▶ httpd ─┬─ /api/* ──▶ simple-api:8080
                           └─ /*     ──▶ front:80
```

`http-server/my-httpd.conf`

```apache
ProxyPass /api/ http://simple-api:8080/
ProxyPassReverse /api/ http://simple-api:8080/
ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/
```

La règle `/api/` est déclarée avant `/`, sinon `/` capturerait toutes les requêtes. Le slash final de `/api/` et de `http://simple-api:8080/` retire le préfixe : `/api/departments` est transmis à l'API comme `/departments`.

### Configuration du front

L'URL de l'API est injectée au moment du build par Vue CLI à partir de `front/.env.production` :

```
VUE_APP_API_URL=nicolas.estermann.takima.school/api
```

Le front appelle alors `http://nicolas.estermann.takima.school/api/departments`, requête reçue par httpd puis redirigée vers l'API.

### CI/CD

Une entrée est ajoutée à la matrice de `build-and-push.yml` :

```yaml
        include:
          - { image: simple-api, context: simple-api }
          - { image: database, context: database }
          - { image: httpd, context: http-server }
          - { image: front, context: front }
```

L'image `tp-devops-front` est construite et poussée sur DockerHub à chaque push. Le job `deploy` dépend de toute la matrice (`needs: build-and-push`), il attend donc la fin des quatre builds avant de déployer.

### Rôle `front`

`ansible/roles/front/tasks/main.yml`

```yaml
- name: Run front
  community.docker.docker_container:
    name: "{{ front_container }}"
    image: "{{ dockerhub_user }}/{{ front_image }}:latest"
    pull: true
    restart_policy: always
    networks:
      - name: app-network
```

Le conteneur n'expose aucun port : il n'est joignable que par httpd via le réseau `app-network`, sous le nom `front`.

Ajouts dans `playbook.yml` :

```yaml
  vars:
    front_image: tp-devops-front
    front_container: front
  roles:
    - network
    - database
    - app
    - front
    - proxy
```

Le rôle `front` est lancé avant `proxy` pour que httpd trouve ses deux backends au démarrage.

### Vérification

- `http://nicolas.estermann.takima.school/` affiche le front ;
- `http://nicolas.estermann.takima.school/api/departments` renvoie le JSON de l'API ;
- la page Departments du front liste IRC, ETI et CGP, ce qui prouve que la chaîne front → httpd → API → base fonctionne.

## Going Further : Continuous Deployment avec Ansible Vault

Jusqu'ici, les identifiants de la base étaient écrits en clair dans `playbook.yml`, donc visibles par toute personne ayant accès au repo. 

### Fichier de secrets

`ansible/group_vars/all/vault.yml`, avant chiffrement :

```yaml
vault_db_user: usr
vault_db_password: pwd
```

Chiffrement :

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Une fois chiffré, le fichier commence par `$ANSIBLE_VAULT;1.1;AES256` suivi du contenu chiffré, illisible sans le mot de passe. Pour le consulter ou le modifier : `ansible-vault view` et `ansible-vault edit`.

### Utilisation dans le playbook

```yaml
  vars:
    db_user: "{{ vault_db_user }}"
    db_password: "{{ vault_db_password }}"
```

Les rôles continuent d'utiliser `db_user` et `db_password` sans aucune modification. Le préfixe `vault_` est la convention recommandée par la documentation Ansible : il indique que la valeur vient du fichier chiffré, et on retrouve facilement où une variable est définie avec un simple `grep`, alors que le contenu du vault, lui, n'est pas lisible.

### Exécution en local

```bash
ansible-playbook -i inventories/setup.yml playbook.yml --ask-vault-pass
```

### Intégration dans la CI

Le mot de passe de vault est stocké dans le secret GitHub `ANSIBLE_VAULT_PASSWORD`. Le job `deploy` l'écrit dans un fichier temporaire sur le runner et le passe à Ansible :

```yaml
      - name: Write SSH key and vault password
        run: |
          printf '%s\n' "${{ secrets.SSH_PRIVATE_KEY }}" | tr -d '\r' > key
          chmod 600 key
          printf '%s' "${{ secrets.ANSIBLE_VAULT_PASSWORD }}" > vault_pass
          chmod 600 vault_pass

      - name: Deploy with Ansible
        working-directory: ansible
        env:
          ANSIBLE_HOST_KEY_CHECKING: "False"
        run: ansible-playbook -i inventories/setup.yml playbook.yml -e ansible_ssh_private_key_file=../key --vault-password-file ../vault_pass
```

| Élément | Rôle |
|---|---|
| `ANSIBLE_VAULT_PASSWORD` | Mot de passe de vault, stocké uniquement dans les secrets GitHub et masqué dans les logs |
| `printf '%s'` | Écrit le mot de passe sans retour à la ligne final, qui ferait partie du mot de passe |
| `chmod 600` | Restreint la lecture du fichier au seul utilisateur du runner |
| `--vault-password-file` | Fournit le mot de passe sans interaction, indispensable en CI |
| `.gitignore` (`.vault_pass*`) | Empêche de commit un fichier de mot de passe par erreur en local |

## Bonus 1 : Load balancing de l'API

L'API tourne désormais en plusieurs instances, et httpd répartit les requêtes entre elles. Si une instance tombe, les autres continuent de répondre.

```
Navigateur ──:80──▶ httpd ─┬─ /api/* ──▶ balancer://api ─┬─▶ simple-api-1:8080
                           │                              └─▶ simple-api-2:8080
                           └─ /*     ──▶ front:80
```

### Rôle `app` : plusieurs instances

`ansible/roles/app/tasks/main.yml`

```yaml
- name: Run backend API instances
  community.docker.docker_container:
    name: "{{ api_container }}-{{ item }}"
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
  loop: "{{ range(1, api_replicas + 1) | list }}"

- name: Remove old single API container
  community.docker.docker_container:
    name: "{{ api_container }}"
    state: absent
```

Avec `api_replicas: 2` dans les `vars` du playbook, la boucle `loop` crée `simple-api-1` et `simple-api-2`.

### Configuration httpd

`http-server/my-httpd.conf`

```apache
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so

<VirtualHost *:80>
ProxyPreserveHost On

<Proxy "balancer://api">
    BalancerMember http://simple-api-1:8080
    BalancerMember http://simple-api-2:8080
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass /api/ balancer://api/
ProxyPassReverse /api/ balancer://api/
ProxyPass / http://front:80/
ProxyPassReverse / http://front:80/
</VirtualHost>
```

| Élément | Rôle |
|---|---|
| `mod_proxy_balancer` | Permet de définir un groupe de backends `balancer://` |
| `mod_lbmethod_byrequests` | Algorithme de répartition *round robin* : les requêtes sont distribuées à tour de rôle |
| `mod_slotmem_shm` | Mémoire partagée utilisée par le balancer pour suivre l'état des membres |
| `BalancerMember` | Une instance de l'API, joignable par son nom sur `app-network` |
| `ProxySet lbmethod=byrequests` | Choix de l'algorithme. Alternatives : `bybusyness` (instance la moins occupée) ou `bytraffic` (volume de données) |

### Vérification

```bash
ansible all -i inventories/setup.yml -m command -a "docker ps" --become --ask-vault-pass
```

`simple-api-1` et `simple-api-2` sont `Up`. Après plusieurs appels à `/api/departments`, les requêtes apparaissent dans les logs des deux instances. En arrêtant une instance (`docker stop simple-api-1`), l'API continue de répondre grâce à la seconde.

## Bonus 2 : Centralisation des logs avec Grafana, Loki et Alloy

Sans outil dédié, consulter les logs impose de se connecter au serveur et de lancer `docker logs` conteneur par conteneur. Avec plusieurs instances de l'API, cela devient vite ingérable. 
Grafana centralise les logs de tous les conteneurs et permet de les filtrer depuis une interface web.

```
Conteneurs ──▶ Alloy ──▶ Loki ──▶ Grafana ◀── navigateur (/grafana/)
 (stdout)    (collecte) (stockage) (affichage)
```

| Composant | Rôle |
|---|---|
| **Alloy** | Agent de collecte. Découvre les conteneurs via le socket Docker et envoie leurs logs à Loki. Successeur de Promtail |
| **Loki** | Base de logs. Indexe les logs par labels (`container`, `job`) plutôt que par contenu, ce qui la rend légère |
| **Grafana** | Interface de visualisation. Interroge Loki avec le langage LogQL |

### Rôle `monitoring`

```
ansible/roles/monitoring/
├── files/
│   ├── config.alloy
│   └── datasources.yml
└── tasks/
    └── main.yml
```

`files/config.alloy`

```
discovery.docker "containers" {
  host = "unix:///var/run/docker.sock"
}

discovery.relabel "containers" {
  targets = discovery.docker.containers.targets

  rule {
    source_labels = ["__meta_docker_container_name"]
    regex         = "/(.*)"
    target_label  = "container"
  }
}

loki.source.docker "default" {
  host       = "unix:///var/run/docker.sock"
  targets    = discovery.relabel.containers.output
  labels     = { "job" = "docker" }
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint {
    url = "http://loki:3100/loki/api/v1/push"
  }
}
```

Alloy découvre tous les conteneurs, transforme leur nom Docker (`/simple-api-1`) en label `container="simple-api-1"`, lit leurs logs et les pousse vers Loki.

`files/datasources.yml`

```yaml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: true
```

Ce fichier de *provisioning* connecte automatiquement Grafana à Loki au démarrage, sans configuration manuelle dans l'interface.

`tasks/main.yml`

```yaml
- name: Create monitoring config directory
  file:
    path: /opt/monitoring
    state: directory

- name: Copy Alloy config
  copy:
    src: config.alloy
    dest: /opt/monitoring/config.alloy

- name: Copy Grafana datasource
  copy:
    src: datasources.yml
    dest: /opt/monitoring/datasources.yml

- name: Run Loki
  community.docker.docker_container:
    name: loki
    image: grafana/loki:latest
    restart_policy: always
    networks:
      - name: app-network
    volumes:
      - loki-data:/loki

- name: Run Alloy
  community.docker.docker_container:
    name: alloy
    image: grafana/alloy:latest
    restart_policy: always
    command: run /etc/alloy/config.alloy
    networks:
      - name: app-network
    volumes:
      - /opt/monitoring/config.alloy:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro

- name: Run Grafana
  community.docker.docker_container:
    name: grafana
    image: grafana/grafana:latest
    restart_policy: always
    networks:
      - name: app-network
    env:
      GF_SECURITY_ADMIN_PASSWORD: "{{ vault_grafana_password }}"
      GF_SERVER_ROOT_URL: "http://nicolas.estermann.takima.school/grafana/"
      GF_SERVER_SERVE_FROM_SUB_PATH: "true"
    volumes:
      - /opt/monitoring/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml:ro
      - grafana-data:/var/lib/grafana
```

| Élément | Rôle |
|---|---|
| `copy` | Dépose les fichiers de configuration du rôle sur le serveur, montés ensuite dans les conteneurs |
| `/var/run/docker.sock:ro` | Donne à Alloy un accès en lecture seule à l'API Docker pour découvrir les conteneurs et lire leurs logs |
| `loki-data`, `grafana-data` | Volumes nommés : les logs et la configuration Grafana survivent aux redéploiements |
| `vault_grafana_password` | Mot de passe admin de Grafana, stocké chiffré dans le vault avec les autres secrets |
| `GF_SERVER_SERVE_FROM_SUB_PATH` | Permet à Grafana de fonctionner derrière httpd sous le chemin `/grafana/` |


### Exposition via httpd

Dans le `VirtualHost`, avant la règle `/` :

```apache
ProxyPass /grafana/ http://grafana:3000/grafana/
ProxyPassReverse /grafana/ http://grafana:3000/grafana/
```

### Playbook

```yaml
  roles:
    - network
    - database
    - app
    - front
    - monitoring
    - proxy
```

### Utilisation

Grafana est accessible sur `http://nicolas.estermann.takima.school/grafana/` (utilisateur `admin`). Dans **Explore**, avec la source Loki, quelques requêtes LogQL :

| Requête | Résultat |
|---|---|
| `{container="simple-api-1"}` | Logs d'une instance de l'API |
| `{container=~"simple-api-.*"}` | Logs des deux instances, ce qui permet d'observer le load balancing |
| `{container="httpd"} \|= "500"` | Requêtes ayant renvoyé une erreur 500 |
| `{job="docker"} \|= "ERROR"` | Toutes les erreurs, tous conteneurs confondus |

Les deux bonus se complètent : le load balancing multiplie les instances, et Grafana permet de suivre leur activité à un seul endroit.
