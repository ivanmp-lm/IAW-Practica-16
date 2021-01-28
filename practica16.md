---
description: >-
  En esta práctica se utilizará un fichero dockerfile para lanzar una aplicación
  en una pila LAMP.
---

# Práctica 16 - «Dockerizar» una aplicación LAMP



**DATOS DE LA MÁQUINA PARA SU REVISIÓN:**

* **IP:** [**http://54.80.221.164/index.php**](http://54.80.221.164/index.php)
* **ACCESO PHPMYADMIN:** [**http://54.80.221.164:8080/**](http://54.80.221.164:8080/)
* **USUARIO PHPMYADMIN:** root
* **CLAVE PHPMYADMIN:** root

Esta práctica empieza igual que las dos anteriores, pero con la peculiaridad de que se utilizará docker-compose combinado con dockerfile para crear nuestro propio contenedor en el que se lanzará una aplicación de Apache conectada a una pila LAMP.

El primer paso será crear el archivo dockerfile ya que el nuevo archivo de docker-compose se servirá de él posteriormente. El resultado del archivo es el siguiente:

```text
FROM ubuntu:focal

LABEL author="José Juan Sánchez"

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update \
    && apt install apache2 -y\
    && apt install php -y\
    && apt install libapache2-mod-php -y\
    && apt install php-mysql -y

RUN apt install git -y \
    && cd /tmp \
    && git clone https://github.com/josejuansanchez/iaw-practica-lamp \
    && mv /tmp/iaw-practica-lamp/src/* /var/www/html \
    && sed -i 's/localhost/mysql/' /var/www/html/config.php \
    && rm /var/www/html/index.html

EXPOSE 80

CMD ["/usr/sbin/apache2ctl", "-D", "FOREGROUND"]
```

El dockerfile utilizará una imagen de ubuntu focal, y se ejecutará el script detallado a continuación, que no es otro que el utilizado en las primeras prácticas del curso. El archivo estará alojado en una carpeta llamada "apache" ya que en el archivo de compose necesitaremos referenciarlo de alguna manera para construirlo, así que es necesario que esté dentro de una carpeta.

El archivo docker-compose utilizado es el siguiente:

```text
version: '3.4'


services:
  mysql:
    image: mysql:5.7.28
    ports: 
      - 3306:3306
    command: --default-authentication-plugin=mysql_native_password
    environment: 
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes: 
      - mysql_data:/var/lib/mysql
      - ./sql:/docker-entrypoint-initdb.d
    networks:
      - backend-network
    restart: always

  phpmyadmin:
    image: phpmyadmin
    ports:
      - 8080:80
    depends_on:
      - "mysql"
    environment: 
      - PMA_ARBITRARY=1
    networks:
      - backend-network
      - frontend-network
    restart: always

  apache:
    build: ./apache
    ports: 
      - 80:80
    depends_on:
      - "mysql"
    networks:
      - backend-network
      - frontend-network
    restart: always

volumes:
  mysql_data:

networks:
  backend-network:
  frontend-network:
```

Nótese en el apartado "apache" que se utiliza la función "build" en lugar de "image", ya que no estamos descargando una imagen de docker hub si no que estamos creando una propia, en esa línea también se puede ver la referencia a la carpeta anteriormente creada a la que se hace referencia para poder usarse el archivo dockerfile.

En la imagen de mysql, se puede ver una línea que no se utilizó en las anteriores prácticas que referencia a un nuevo volumen, la línea "./sql:/docker-entrypoint-initdb.d" que servirá para que el contenedor de apache con la aplicación web pueda conectarse con mysql.

Por último, el archivo .env que no difiere con el del resto de prácticas exceptuando que los datos son diferentes para adaptarse a la aplicación:

```text
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=lamp_db
MYSQL_USER=lamp_user
MYSQL_PASSWORD=lamp_password
```

Con estos componentes, se lanzarán los contenedores con el comando de siempre:

```text
$ sudo docker-compose up -d
```

Tras esto, debería ser posible acceder a la aplicación LAMP utilizando la IP pública del contenedor:

![](https://raw.githubusercontent.com/ivanmp-lm/IAW/master/.gitbook/assets/image%20(23).png)

No obstante, no se podrán inscribir datos aún, se deberá acceder a phpMyAdmin por el puerto 8080 y ejecutar el siguiente script \(utilizado también en las primeras prácticas\):

```text
DROP DATABASE IF EXISTS lamp_db;
CREATE DATABASE lamp_db CHARSET utf8mb4;
USE lamp_db;

CREATE TABLE users (
  id int(11) NOT NULL auto_increment,
  name varchar(100) NOT NULL,
  age int(3) NOT NULL,
  email varchar(100) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE USER IF NOT EXISTS 'lamp_user'@'%';
SET PASSWORD FOR 'lamp_user'@'%' = 'lamp_password';
GRANT ALL PRIVILEGES ON lamp_db.* TO 'lamp_user'@'%';
```

![](https://raw.githubusercontent.com/ivanmp-lm/IAW/master/.gitbook/assets/image%20(24).png)

Tras ejecutar el script, se podrá agregar información a la aplicación y se dará por concluida la práctica:

![](https://raw.githubusercontent.com/ivanmp-lm/IAW/master/.gitbook/assets/image%20(33).png)
