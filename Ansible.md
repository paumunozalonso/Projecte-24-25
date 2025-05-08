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
