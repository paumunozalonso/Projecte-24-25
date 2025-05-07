# Instal·lació i configuració de Zabbix


---

## Creació de la base de dades per a Zabbix

Primer, vam accedir a MySQL per crear una base de dades anomenada `zabbix`, on es guardarà tota la informació relacionada amb el monitoratge.

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'Hola1234!';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
FLUSH PRIVILEGES;
```

---

## Afegir el repositori oficial de Zabbix

Per instal·lar Zabbix, primer vam afegir el repositori oficial:

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
```

---

## Instal·lació dels paquets de Zabbix

Un cop afegit el repositori, instal·lem els paquets següents:

```bash
apt install zabbix-agent zabbix-server-mysql php-mysql php-sql zabbix-sql-scripts zabbix-apache-conf
```

---

## Importació de l’estructura de la base de dades

Executem la comanda per importar l’estructura SQL al nou esquema `zabbix`:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -p zabbix
```

---

## Configuració del servidor Zabbix

Editem el fitxer `/etc/zabbix/zabbix_server.conf` per indicar-li les dades de connexió a la base de dades:

```conf
DBName=zabbix
DBUser=zabbix
DBPassword=Hola1234!
```

---

## Configuració de l'agent Zabbix

També cal modificar el fitxer `/etc/zabbix/zabbix_agentd.conf` perquè l’agent apunti correctament al servidor Zabbix:

```conf
Server=127.0.0.1
Hostname=Zabbix server
```

---

## Iniciar i activar serveis

Finalment, reiniciem els serveis i els activem perquè s’iniciïn amb el sistema:

```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
```

---


## Demos Zabbix

A continuació es mostren alguns vídeos que mostren el funcionament i les funcionalitats configurades amb Zabbix:

### Funcionament correcte del sistema
[Enllaç al vídeo](https://drive.google.com/file/d/1Yr6WRf7TLc8tqSaOMcnGgSdwqRbC_atV/view?usp=sharing)

### Informe generat per Zabbix
[Enllaç al vídeo](https://drive.google.com/file/d/1uWL6xegRA5jkvzS_l1tSv0KJ0ty1Po9N/view?usp=sharing)

### Notificacions sobre incidències

- [Notificació 1](https://drive.google.com/file/d/1bEAZo4PhiArJEVmRNc6N1qyZDgqctVl6/view?usp=sharing)
- [Notificació 2](https://drive.google.com/file/d/1qHwMuufDFeJsC7v-NjA_Ba0OwPaJPjkn/view?usp=sharing)



