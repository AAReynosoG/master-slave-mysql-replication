# Configuración de Replicación Master-Slave en MySQL

Práctica realizada el día `01-04-2025`, utilizando `Rocky Linux 9` y desplegada en `DigitalOcean`, como parte de la asignatura `Seguridad en el Desarrollo de Aplicaciónes`.

Funcional respecto a los requisitos especificados en el momento de su elaboración.

## 1. Instalación de MySQL

Asegúrate de tener instalado `mysql-server` en ambos Droplets:

```bash
sudo dnf install mysql-server
```

## 2. Configuración del Droplet "Master"

Edita el archivo de configuración de MySQL en el Droplet "Master":

```bash
sudo nano /etc/my.cnf.d/mysql-server.conf
```

Agrega o modifica las siguientes líneas:

```ini
port=13306
bind-address = 10.124.0.2  # IP del Droplet "Master"
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
```

## 3. Habilitar el puerto en SELinux

Ejecuta el siguiente comando para permitir el puerto de MySQL en SELinux:

```bash
sudo semanage port -a -t mysqld_port_t -p tcp 13306
```

## 4. Obtener el estado del Master

Ingresa a MySQL en el Droplet "Master":

```bash
mysql -u root -p
```

Ejecuta el siguiente comando para obtener el estado del Master:

```sql
SHOW MASTER STATUS;
```

Se mostrará una tabla como esta:

```plaintext
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |     1227 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```

Guarda los valores de `File` y `Position`.

## 5. Crear el usuario de replicación

En el Droplet "Master", crea un usuario para la replicación y asigna permisos:

```sql
CREATE USER 'replica'@'10.124.0.3' IDENTIFIED BY 'tuSuperContraseña';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'10.124.0.3';
FLUSH PRIVILEGES;
```
Es importante mencionar que la `IP` con la que se crea el usuario es la del Droplet "Slave".

## 6. Configuración del Droplet "Slave"

Edita el archivo de configuración de MySQL en el Droplet "Slave":

```bash
sudo nano /etc/my.cnf.d/mysql-server.conf
```

Agrega o modifica las siguientes líneas:

```ini
port=16603
bind-address = 10.124.0.3  # IP del Droplet "Slave"
server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
```

## 7. Habilitar los puertos en SELinux

Ejecuta los siguientes comandos en el Droplet "Slave":

```bash
sudo semanage port -a -t mysqld_port_t -p tcp 13306
sudo semanage port -a -t mysqld_port_t -p tcp 16603
```

## 8. Configurar la replicación en el "Slave"

Ingresa a MySQL en el Droplet "Slave":

```bash
mysql -u root -p
```

Ejecuta los siguientes comandos:

```sql
STOP SLAVE;
CHANGE MASTER TO
    MASTER_HOST='10.124.0.2',
    MASTER_USER='replica',
    MASTER_PASSWORD='tuSuperContraseña',
    MASTER_PORT=13306,
    MASTER_LOG_FILE='mysql-bin.000003',
    MASTER_LOG_POS=1227,
    MASTER_SSL=1,
    MASTER_SSL_CA='/etc/mysql/master_certs/ca.pem',
    MASTER_SSL_CERT='/etc/mysql/master_certs/client-cert.pem',
    MASTER_SSL_KEY='/etc/mysql/master_certs/client-key.pem';
START SLAVE;
```
Los atributos `MASTER_HOST` y `MASTER_PORT` deben coincidir con la IP y puerto del Droplet "Master". 

Los valores de `MASTER_LOG_FILE` y `MASTER_LOG_POS` deben coincidir con los valores obtenidos en el paso 4.

## 9. Verificar el estado de la replicación

Ejecuta el siguiente comando en el Droplet "Slave":

```sql
SHOW SLAVE STATUS\G;
```

Verifica que los siguientes valores sean correctos:

```plaintext
Slave_IO_Running: Yes           --> El hilo de I/O está activo y conectado al maestro
Slave_SQL_Running: Yes          --> El hilo SQL está aplicando los cambios
Seconds_Behind_Master: 0        --> No hay retraso en la replicación
Last_IO_Errno: 0                --> Sin errores de conexión
Last_SQL_Errno: 0               --> Sin errores SQL
```

## 10. Prueba de replicación

Para comprobar que la replicación está funcionando, inserta un registro en el Droplet "Master":

```sql
USE test_db;
CREATE TABLE prueba (id INT PRIMARY KEY AUTO_INCREMENT, nombre VARCHAR(50));
INSERT INTO prueba (nombre) VALUES ('Ejemplo');
```

Luego, en el Droplet "Slave", verifica si el registro se ha replicado:

```sql
SELECT * FROM test_db.prueba;
```

Si los datos aparecen en el "Slave", la replicación ha sido configurada exitosamente. 🎉

## 11. Ejemplo Read-Write en Laravel 9

`config/database.php`
```php
        'mysql' => [
            'driver' => 'mysql',
            'sticky' => true,

            'write' => [
                'host' => env('DB_MASTER_HOST'),
                'port' => env('DB_MASTER_PORT'),
                'username' => env('DB_MASTER_USERNAME'),
                'password' => env('DB_MASTER_PASSWORD'),
            ],

            'read' => [
                'host' => env('DB_SLAVE_HOST'),
                'port' => env('DB_SLAVE_PORT'),
                'username' => env('DB_SLAVE_USERNAME'),
                'password' => env('DB_SLAVE_PASSWORD'),
            ],

            'url' => env('DATABASE_URL'),
            'database' => env('DB_DATABASE', 'forge'),
            'unix_socket' => env('DB_SOCKET', ''),
            'charset' => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
            'prefix' => '',
            'prefix_indexes' => true,
            'strict' => true,
            'engine' => null,
            'options' => extension_loaded('pdo_mysql') ? array_filter([
                PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
                PDO::MYSQL_ATTR_SSL_CERT => env('MYSQL_ATTR_SSL_CERT'),
                PDO::MYSQL_ATTR_SSL_KEY => env('MYSQL_ATTR_SSL_KEY'),
                PDO::MYSQL_ATTR_SSL_VERIFY_SERVER_CERT => true,
            ]) : [],
        ],
```
