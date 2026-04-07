## 1. Crear usuario en MySQL para el exporter
``` bash
mysql -u root -p
```

``` sql
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'TU_CONTRASEÑA_SEGURA';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```
## 2. Descargar y extraer el exporter
```bash
cd /tmp
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.16.0/mysqld_exporter-0.16.0.linux-amd64.tar.gz
tar xzf mysqld_exporter-0.16.0.linux-amd64.tar.gz
sudo cp mysqld_exporter-0.16.0.linux-amd64/mysqld_exporter /usr/local/bin/
sudo chmod +x /usr/local/bin/mysqld_exporter
```

## 3. Crear archivo de credenciales
``` bash 
sudo mkdir -p /etc/prometheus
sudo nano /etc/prometheus/.mysqld_exporter.cnf
```

``` ini
[client]
user=exporter
password=TU_CONTRASEÑA_SEGURA
```

#### asignar permisos
```bash
sudo chown root:root /etc/prometheus/.mysqld_exporter.cnf
sudo chmod 600 /etc/prometheus/.mysqld_exporter.cnf
```

## 4. Crear web-config.yml para mysqld_exporter para contraseña web
``` bash
sudo nano /etc/prometheus/mysqld_exporter-web.yml
```

``` yaml
basic_auth_users:
  prometheus: '$2y$12$TU_HASH_AQUI'
```

#### permisos
```bash
sudo chmod 600 /etc/prometheus/mysqld_exporter-web.yml
```

## 5. Crear servicio systemd

```bash
sudo nano /etc/systemd/system/mysqld_exporter.service
```

``` ini 
[Unit]
Description=MySQL Exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/mysqld_exporter \
    --config.my-cnf=/etc/prometheus/.mysqld_exporter.cnf \
    --web.config.file=/etc/prometheus/mysqld_exporter-web.yml \
    --web.listen-address=:9104
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
#### Iniciar
``` bash
sudo systemctl daemon-reload
sudo systemctl enable --now mysqld_exporter
sudo systemctl status mysqld_exporter --no-pager
```
