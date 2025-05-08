# Instal·lació i configuració de Docker


## Configuració inicial de l’entorn


## Instal·lació de Docker


```bash
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc


echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null


sudo apt-get update


sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


curl -SL https://github.com/docker/compose/releases/download/v2.33.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

```


## Fitxer `docker-compose.yml` amb Docker Compose

```yaml
services:
  web:
    image: php:8.2-apache
    container_name: apache_php
    restart: always
    volumes:
      - ./www:/var/www/html
    ports:
      - "8080:80"
    depends_on:
      - db
    networks:
      - lamp-network
    command: >
          /bin/bash -c "apt-get update && docker-php-ext-install mysqli pdo_mysql && docker-php-ext-enable mysqli pdo_mysql && apache2-foreground"

  db:
    image: mysql:5.7
    container_name: mysql_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: hola
      MYSQL_DATABASE: projecte
      MYSQL_USER: paumunoz
      MYSQL_PASSWORD: hola
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - lamp-network

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    restart: always
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      MYSQL_ROOT_PASSWORD: hola
    depends_on:
      - db
    networks:
      - lamp-network

volumes:
  db_data:

networks:
  lamp-network:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.50.0/24
```
# Migració del projecte multi-contenidor de Compose al clúster Swarm

```bash
docker stack deploy -c docker-compose.yml stack_docker


services:
  web:
    image: php:8.2-apache-buster
    volumes:
      - ./www:/var/www/html
    ports:
      - "8080:80"
    depends_on:
      - db
    networks:
      - lamp-network
    command: >
      /bin/bash -c "apt-get update && docker-php-ext-install mysqli pdo_mysql && docker-php-ext-enable mysqli pdo_mysql && apache2-foreground"

  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: hola
      MYSQL_DATABASE: projecte
      MYSQL_USER: paumunoz
      MYSQL_PASSWORD: hola
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - lamp-network

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
      MYSQL_ROOT_PASSWORD: hola
    depends_on:
      - db
    networks:
      - lamp-network

volumes:
  db_data:
  www_data:

networks:
  lamp-network:
    driver: overlay
```

## Creació del Deployment per a cada servei


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.35.0 (9532cef3)
  labels:
    io.kompose.service: web
  name: web
spec:
  replicas: 1
  selector:
    matchLabels:
      io.kompose.service: web
  strategy:
    type: Recreate
  template:
    metadata:
      annotations:
        kompose.cmd: kompose convert
        kompose.version: 1.35.0 (9532cef3)
      labels:
        io.kompose.service: web
    spec:
      containers:
        - args:
            - /bin/bash
            - -c
            - apt-get update && docker-php-ext-install mysqli pdo_mysql && docker-php-ext-enable mysqli pdo_mysql && apache2-foreground
          image: php:8.2-apache-buster
          name: apache_php
          ports:
            - containerPort: 80
              protocol: TCP
          volumeMounts:
            - mountPath: /var/www/html
              name: web-cm0
      restartPolicy: Always
      volumes:
        - configMap:
            name: web-cm0
          name: web-cm0

```
## Fitxer Service per exposar el Deployment

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kompose.cmd: kompose convert
    kompose.version: 1.35.0 (9532ceef3)
  labels:
    io.kompose.service: web
  name: web
spec:
  type: NodePort
  ports:
    - name: "8080"
      port: 8080
      targetPort: 80
      nodePort: 32080
  selector:
    io.kompose.service: web
```

## Instal·lació de Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

minikube start --driver=docker
```
