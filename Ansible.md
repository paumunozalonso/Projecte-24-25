## Ansible fitxers

## `healthy.yml`
```yaml
- name: Comprovar estat de WordPress
  hosts: wordpress_host
  become: true
  tasks:
    - name: Comprovar que WordPress respon
      uri:
        url: http://localhost:8090
        return_content: yes
      register: wordpress_resposta
      delegate_to: wordpress_host

    - name: Validar resposta de WordPress
      assert:
        that:
          - wordpress_resposta.status == 200
          - wordpress_resposta.content | length > 0
        fail_msg: "WordPress no està responent correctament"
        success_msg: "WordPress està funcionant correctament"

- name: Comprovar estat de Nextcloud
  hosts: nextcloud_host
  become: true
  tasks:
    - name: Comprovar que Nextcloud respon
      uri:
        url: http://192.168.1.35:8081
        return_content: yes
      register: nextcloud_resposta
      delegate_to: nextcloud_host

    - name: Validar resposta de Nextcloud
      assert:
        that:
          - nextcloud_resposta.status == 200
          - nextcloud_resposta.content | length > 0
        fail_msg: "Nextcloud no està responent correctament"
        success_msg: "Nextcloud està funcionant correctament"


```


## `inventari.ini`
```yaml
[wordpress]
wordpress_host ansible_connection=local

[nextcloud]
nextcloud_host ansible_host=192.168.1.35 ansible_user=paumunoz ansible_become=true


```


## `playbook.yml`
```yaml

- name: Desplegar WordPress
  hosts: wordpress_host
  become: true
  roles:
    - wordpress

- name: Desplegar Nextcloud
  hosts: nextcloud_host
  become: true
  roles:
    - nextcloud

```

## `remove.yml`
```yaml

- name: Eliminar contenidors, volums i xarxes
  hosts: all
  become: true
  tasks:

    - name: Aturar contenidor WordPress (si existeix)
      community.docker.docker_container:
        name: wordpress_app
        state: absent
      ignore_errors: true

    - name: Aturar contenidor MySQL WordPress (si existeix)
      community.docker.docker_container:
        name: wordpress_db
        state: absent
      ignore_errors: true

    - name: Eliminar volum HTML WordPress (si existeix)
      community.docker.docker_volume:
        name: wordpress_html_volume
        state: absent
      ignore_errors: true

    - name: Eliminar volum DB WordPress (si existeix)
      community.docker.docker_volume:
        name: wordpress_db_volume
        state: absent
      ignore_errors: true

    - name: Eliminar xarxa WordPress (si existeix)
      community.docker.docker_network:
        name: wordpress_net
        state: absent
      ignore_errors: true

    - name: Aturar contenidor Nextcloud (si existeix)
      community.docker.docker_container:
        name: nextcloud_app
        state: absent
      ignore_errors: true

    - name: Aturar contenidor MariaDB Nextcloud (si existeix)
      community.docker.docker_container:
        name: nextcloud_db
        state: absent
      ignore_errors: true

    - name: Eliminar volum HTML Nextcloud (si existeix)
      community.docker.docker_volume:
        name: nextcloud_html_volume
        state: absent
      ignore_errors: true

    - name: Eliminar volum DB Nextcloud (si existeix)
      community.docker.docker_volume:
        name: nextcloud_db_volume
        state: absent
      ignore_errors: true

    - name: Eliminar xarxa Nextcloud (si existeix)
      community.docker.docker_network:
        name: nextcloud_net
        state: absent
      ignore_errors: true

```

## `start.yml`
```yaml

- name: Arrencar contenidors
  hosts: all
  become: true
  tasks:
    - name: Arrencar contenidor WordPress
      community.docker.docker_container:
        name: wordpress_app
        state: started
      when: inventory_hostname == "wordpress_host"

    - name: Arrencar contenidor MySQL de WordPress
      community.docker.docker_container:
        name: wordpress_db
        state: started
      when: inventory_hostname == "wordpress_host"

    - name: Arrencar contenidor Nextcloud
      community.docker.docker_container:
        name: nextcloud_app
        state: started
      when: inventory_hostname == "nextcloud_host"

    - name: Arrencar contenidor MariaDB de Nextcloud
      community.docker.docker_container:
        name: nextcloud_db
        state: started
      when: inventory_hostname == "nextcloud_host"

```


## `stop.yml`
```yaml
- name: Aturar contenidors
  hosts: all
  become: true
  tasks:

    - name: Aturar WordPress (contenidor app)
      community.docker.docker_container:
        name: wordpress_app
        state: stopped
      when: "'wordpress' in group_names"

    - name: Aturar WordPress (contenidor DB)
      community.docker.docker_container:
        name: wordpress_db
        state: stopped
      when: "'wordpress' in group_names"

    - name: Aturar Nextcloud (contenidor app)
      community.docker.docker_container:
        name: nextcloud_app
        state: stopped
      when: "'nextcloud' in group_names"

    - name: Aturar Nextcloud (contenidor DB)
      community.docker.docker_container:
        name: nextcloud_db
        state: stopped
      when: "'nextcloud' in group_names"


```

## Nextcloud

## `handlers/main.yml`
```yaml
- name: Reiniciar contenidor Nextcloud
  community.docker.docker_container:
    name: nextcloud_app
    state: started
    restart: true


- name: Reiniciar contenidor MariaDB
  community.docker.docker_container:
    name: nextcloud_db
    state: restarted

```

## `vars/main.yml`
```yaml
db_user: user
db_password: pass
db_root_password: root

```


## `tasks/main.yml`
```yaml
- name: Instal·lar Python i pip
  apt:
    name:
      - python3
      - python3-pip
    state: present
    update_cache: true

- name: Instal·lar Docker SDK per a Python
  apt:
    name: python3-docker
    state: present

- name: Crear xarxa Docker
  community.docker.docker_network:
    name: nextcloud_net

- name: Actualitzar imatge de Nextcloud
  community.docker.docker_image:
    name: nextcloud:latest
    source: pull
  notify: Reiniciar contenidor Nextcloud

- name: Actualitzar imatge de MariaDB
  community.docker.docker_image:
    name: mariadb:latest
    source: pull
  notify: Reiniciar contenidor MariaDB

- name: Crear volum per dades de Nextcloud
  community.docker.docker_volume:
    name: nextcloud_html_volume

- name: Crear volum per dades de MariaDB
  community.docker.docker_volume:
    name: nextcloud_db_volume

- name: Desplegar contenidor MariaDB
  community.docker.docker_container:
    name: nextcloud_db
    image: mariadb:latest
    state: started
    restart_policy: always
    env:
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: "{{ db_user }}"
      MYSQL_PASSWORD: "{{ db_password }}"
      MYSQL_ROOT_PASSWORD: "{{ db_root_password }}"
    volumes:
      - nextcloud_db_volume:/var/lib/mysql
    networks:
      - name: nextcloud_net

- name: Desplegar contenidor Nextcloud
  community.docker.docker_container:
    name: nextcloud_app
    image: nextcloud:latest
    state: started
    restart_policy: always
    ports:
      - "8081:80"
    env:
      MYSQL_PASSWORD: "{{ db_password }}"
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: "{{ db_user }}"
      MYSQL_HOST: nextcloud_db
    volumes:
      - nextcloud_html_volume:/var/www/html
    networks:
      - name: nextcloud_net

- import_tasks: security.yml

```

## ` tasks/security.yml`
```yaml
- name: Instal·lar UFW
  apt:
    name: ufw
    state: present
    update_cache: true

- name: Activar UFW i denegar tràfic
  community.general.ufw:
    state: enabled
    policy: deny


- name: Permetre tràfic SSH (port 22)
  community.general.ufw:
    rule: allow
    port: 22
    proto: tcp

- name: Permetre accés a Nextcloud només des de 192.168.1.0/24
  community.general.ufw:
    rule: allow
    port: 8081
    proto: tcp
    src: 192.168.1.0/24
```

## Wordpress

## `handlers/main.yml`
```yaml
- name: Reiniciar contenidor WordPress
  community.docker.docker_container:
    name: wordpress_app
    state: restarted

- name: Reiniciar contenidor MySQL
  community.docker.docker_container:
    name: wordpress_db
    state: restarted
```

## `vars/main.yml`
```yaml
mysql_user: user
mysql_password: pass
mysql_root_password: root
```

## `tasks/main.yml`
```yaml
- name: Instal·lar Python i pip
  apt:
    name:
      - python3
      - python3-pip
    state: present
    update_cache: true

- name: Instal·lar Docker SDK per a Python
  apt:
    name: python3-docker
    state: present

- name: Crear xarxa Docker
  community.docker.docker_network:
    name: wordpress_net

- name: Actualitzar imatge de WordPress
  community.docker.docker_image:
    name: wordpress:latest
    source: pull
  notify: Reiniciar contenidor WordPress

- name: Actualitzar imatge de MySQL
  community.docker.docker_image:
    name: mysql:5.7
    source: pull
  notify: Reiniciar contenidor MYSQL

- name: Crear volum per WordPress HTML
  community.docker.docker_volume:
    name: wordpress_html_volume

- name: Crear volum per dades de MySQL
  community.docker.docker_volume:
    name: wordpress_db_volume

- name: Desplegar contenidor MySQL
  community.docker.docker_container:
    name: wordpress_db
    image: mysql:5.7
    state: started
    restart_policy: always
    env:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: "{{ mysql_user }}"
      MYSQL_PASSWORD: "{{ mysql_password }}"
      MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
    volumes:
      - wordpress_db_volume:/var/lib/mysql
    networks:
      - name: wordpress_net

- name: Desplegar contenidor WordPress
  community.docker.docker_container:
    name: wordpress_app
    image: wordpress:latest
    state: started
    restart_policy: always
    ports:
      - "8090:80"
    env:
      WORDPRESS_DB_HOST: wordpress_db
      WORDPRESS_DB_USER: "{{ mysql_user }}"
      WORDPRESS_DB_PASSWORD: "{{ mysql_password }}"
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_html_volume:/var/www/html
    networks:
      - name: wordpress_net

- import_tasks: security.yml
```

## `tasks/security.yml`
```yaml
- name: Instal·lar UFW
  apt:
    name: ufw
    state: present
    update_cache: true

- name: Denegar tot altre tràfic i activar UFW
  community.general.ufw:
    state: enabled
    policy: deny


- name: Permetre tràfic SSH (port 22)
  community.general.ufw:
    rule: allow
    port: 22
    proto: tcp

- name: Permetre accés a WordPress només des de 192.168.1.0/24
  community.general.ufw:
    rule: allow
    port: 8090
    proto: tcp
    src: 192.168.1.0/24
```

## Demos


- **Playbook principal**  
  [https://drive.google.com/file/d/1Rd6pWVf7t5tOCbnoqBw6iz8dZ0Si_quj/view?usp=sharing](https://drive.google.com/file/d/1Rd6pWVf7t5tOCbnoqBw6iz8dZ0Si_quj/view?usp=sharing)

- **Tasques individuals per contenidors**  
  [https://drive.google.com/file/d/1pTcxKHu2VHWpLsvdn4BdRy25VWIPURu7/view?usp=sharing](https://drive.google.com/file/d/1pTcxKHu2VHWpLsvdn4BdRy25VWIPURu7/view?usp=sharing)

- **Comprovacions**  
  [https://drive.google.com/file/d/1OMxTKTkz_up-TNxGOPdV88NXKiKopmiF/view?usp=sharing](https://drive.google.com/file/d/1OMxTKTkz_up-TNxGOPdV88NXKiKopmiF/view?usp=sharing)
