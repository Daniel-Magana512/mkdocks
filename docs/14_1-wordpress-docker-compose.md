# Practica 14.1 Instalación de WordPress usando contenedores Docker y Docker Compose

## **Para la creación de la Instancia**

Voy a usar terraform debido a su sencillez y que requiere de menor tiempo a la hora de generar grupos de seguridad , instancia y demás.

**Importante** voy a tener los puertos :

* **80** y el **443** lo utilizará la imagen de https-portal.
* **8080** Es el puerto que utilizaremos para el phpmyadmin y así conectarnos.
* **22** Nos conectaremos por ssh.

Para ejecutar el archivo tendremos que escribir:


```command
terraform init
```

EL comando mencionado nos instalará los módulos necesarios para ejecutar el archivo.

EL siguiente comandonos permitirá lo que es el archivo (recordar, que para ejecutar este comando como el anterior hay que estar dentro del directorio):

```command
terraform apply
```

Contenido del archivo es el siguiente:

```terraform
#Configuramos el proveedor de AWS y le ponemos la región.

provider "aws" {
  region = "us-east-1"
}

#---------------------------------------------------------------------------------------------------------------------------------
#Creamos la maquina junto con sus recursos
#-------------------------------------------------------------------------------------------------------------------------------------------

# Creamos un grupo de seguridad, especificando las condiciones que tendrá.
#No hay que preocuparse por nivelarlas la información por el comando terraform fmt.
#En aws_security_group (este parametro es un recurso, por lo tanto debe de ser la misma) le decimos que va a ser un grupo de seguridad.

resource "aws_security_group" "sg_practica14_1" {
  name        = "sg_practica14_1"
  description = "Grupo de seguridad para la instancia de sg_practica14_1"

  #Especificamos los puertos y su condiciones

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
#Las siguientes lineas son para las reglas de salida (son importante ya que de lo contrario no podría por ejemplo actualizar repositorios)
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Creamos la instancia
#La primera línea es para declarar lo que vamos a hacer.
#Después le pasamos los elementos ensenciales como la ami, 
#su capacidad, la clave shh y asíganamos la instancia al grupo.


resource "aws_instance" "practica14_1" {
  ami             = "ami-00874d747dde814fa"
  instance_type   = "t2.large"
  key_name        = "vockey"

#En security_groups especificamos el grupo de seguridad.

  security_groups = [aws_security_group.sg_practica14_1.name]

  #En la siguiente línea ponemos el nombre a la instancia.
  tags = {
    Name = "practica14_1"
  }
}

# Creamos una IP elástica. En este apartado hago referencia 
#al nombre de la instancia. 

resource "aws_eip" "ip_elastica" {
  instance = aws_instance.practica14_1.id
}

# Mostramos la IP pública de la instancia
output "elastic_ip" {
  value = aws_eip.ip_elastica.public_ip
}
```
## **Apartado de las imágenes Docker**

Para ello voy a utilizar un archivo llamado docker-compose.yml y tendrá un archivo .env (que tendrá el contenido de las variables).

```.env
# WordPress variables

#Nombre de la base de datos.
WORDPRESS_DB_NAME=wp_database

#Nombre del usuario
WORDPRESS_DB_USER=wp_user

#NOmbre de la contraseña
WORDPRESS_DB_PASSWORD=wp_password

# MySQL Server environment variables
MYSQL_ROOT_PASSWORD=root
```

El contenido del archivo docker-compose.yml:

```yml

version: '3'

#En services declaramos las imágenes y demás.

services:

#Instalación de la imágen de wordpress esta imágen está sacada de docker hub, si yo en image pongo wordpress por defecto me cogerá la version latest. (Me cogerá siempre la última el problema de esto que puede salir una versión que no sea compatible, por eso pongo una versión concreta)

  wordpress:
    image: wordpress:6.1
    #En ports pongo los puertos , en esta imagen como más adelante voy a usar https por eso no asignaré los puerto a wordpress ya que podría dar problemas.
#    ports:
#      - 80:80
    environment:
    #En las siguientes lineas usaremos las variables : (importante, para declarar las variables hay que leer las instrucciones de como funcionan dichas variables.)

    #Ponemos el nombre del servicio de mysql.
    #Asignamos el nombre a la base de datos
    #Asignamos el nombre del usuario para la base de datos.
    #Asignamos contraseña a dicho usuario
      - WORDPRESS_DB_HOST=mysql
      - WORDPRESS_DB_NAME=${WORDPRESS_DB_NAME}
      - WORDPRESS_DB_USER=${WORDPRESS_DB_USER}
      - WORDPRESS_DB_PASSWORD=${WORDPRESS_DB_PASSWORD}
    volumes: 
    #Montamos el volumen, esta dirección hay que consultarlo en docker hub (wordpress), debido a que no puedo poner la ruta que yo quiera sino la que me diga en las instrucciones, de lo contrario no funcionará.
    #Si pongo wordpress_data no compartirá nada de mi ordenador a la imagen, si yo quisiera que usará archivos locales pondría ./ruta_del_archivo.
      - wordpress_data:/var/www/html
    depends_on:
    #Para indicarle que se inicie cuando la imagen de mysql, se inicie y además cuando mysql funcione bien
      mysql:
        condition: service_healthy
    #En caso de error que reinicie siempre.
    restart: always
    #Declaramos las redes a las que pertenece (wordpress necesita estar en las dos debido a que necesitará tener acceso a mysql (backend-network) y al exterior para ofrecer contenido al exterior(frontend-network))
    networks: 
      - frontend-network
      - backend-network

#Para la imagen de mysql.

#En imagen ponemos la versión que queremos descargar en mi caso es mysql:8.0

  mysql:
    image: mysql:8.0
    ports:
    #Ponemos el puerto que usará, el puerto 3306 lo voy a comentar por seguridad, ya que no quiero que cualquiera pueda acceder.
#      - 3306:3306
    environment:
    
    #Usamos las variables para:

    #Usar contraseña de root.
    #Asignamos el nombre a la base de datos
    #Asignamos el nombre del usuario para la base de datos.
    #Asignamos contraseña a dicho usuario
    
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${WORDPRESS_DB_NAME}
      - MYSQL_USER=${WORDPRESS_DB_USER}
      - MYSQL_PASSWORD=${WORDPRESS_DB_PASSWORD}
    volumes:
    #Generamos el volumen, y nos fijamos que la ruta que hemos puesto es la que viene en las instrucciones.
      - mysql_data:/var/lib/mysql
    #En caso de que haya algún error se reiniciará siempre.  
    restart: always
    #Importante:
    # El healthcheck, sirve para garantizar que el servicio de MySQL está listo para aceptar conexiones, si yo pusiera depends (y el servicio) significaría como ese este iniciado, healthcheck no es cuando se haya iniciado sino que se produce hasta que al fin se haya comprobado que los parametros hayan transcurrido con exito.

    #En este caso me comrpueba el usuario y la contraseña
    healthcheck:
      test: mysql --user=${WORDPRESS_DB_USER} --password=${WORDPRESS_DB_PASSWORD} -e "SHOW DATABASES;"
    #interval: Especifica la frecuencia con la que se ejecutará el healthcheck. En este caso, se está especificando un intervalo de 6 segundos.

    #timeout: Especifica el tiempo máximo que se permitirá para que el healthcheck complete. En este caso, se está especificando un timeout de 3 segundos.
    
    #retries: Especifica el número de intentos que se realizarán si el healthcheck falla. En este caso, se está especificando 5 intentos.
      interval: 6s
      timeout: 3s
      retries: 5
    networks: 
    #Añadimos la red
      - backend-network

#Ahora usaremos la imagen phpmyadmin la version 5.2
  phpmyadmin:
    image: phpmyadmin:5.2
    ports:
    #NOs conectaremos por el puerto 8080
      - 8080:80
    environment: 
      - PMA_HOST=mysql
      #Ponemos que se inicie cuando funcione perfectamente mysql
    depends_on:
      mysql:
        condition: service_healthy
    #Reiniciamos siempre
    restart: always
    #Phpmyadmin debe de estar en las dos redes.
    networks: 
      - frontend-network
      - backend-network

#Usaremos la imagen https-portal version 1, importante aclarar que no es oficial, esta imagen está desarrollada por steveltn

  https-portal:
    image: steveltn/https-portal:1
    ports:
    #Los puertos que trabajará son los siguientes:
      - 80:80
      - 443:443

    #Montamos el volume en /var/lib/https-portal
    volumes:
      - ssl_certs_data:/var/lib/https-portal
    environment:
    #He puesto dos tipos de dominios, no pueden ser dos a la vez, la primera se usará de forma local y la otra es para acceder a internet, el dominio que pongo tiene que estar registrado en no-ip.
    # http://wordpress:80 es wordpress debido al nombre del servicio, si tuviese otro nombre cambiariamos wordpress por el nombre del servicio que actue por wordpress

#      DOMAINS: 'localhost -> http://wordpress:80 #local'
      DOMAINS: 'dmp-test.ddns.net -> http://wordpress:80 #production'
    depends_on:
    #Ponemos que este servicio se monte cuando wordpress haya iniciado
      - wordpress
    #Reiniciamos siempre
    restart: always
    #Estará en la red frontend-network , no necesita estar en backend-network, ya que esta wordpress.
    networks:
      - frontend-network

#Tenemos que volver a declarar la ruta de los volumenes.

volumes:
  mysql_data:
  wordpress_data:
  ssl_certs_data:

#Tenemos que volver a declarar el nombre de las redes.

networks:
  frontend-network:
  backend-network:
```

Ya tendremos creada la instancia y el archivo docker-compose.yml con las imagenes.

Faltaría ejecutar el archivo docker-compose.yml y sus herramintas:

## **Ejecutar los servicios e instalar las herramientas necesarias para el proceso**

```yml
---
- name: Playbook para instalar preparar los servicios de la maquina docker
  hosts: maquina
  become: yes

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Instalar el docker.io
      apt:
        name: docker.io
        state: present

    - name: Instalamos pip
      apt:
        name: python3-pip
        state: present

    - name: Instalar el docker
      pip:
        name: docker
        state: present
      
    - name: Instalar el docker-compose
      pip:
        name: docker-compose
        state: present


    - name: Creamos el directorio
      file:
        path: /home/ubuntu/.docker/cli-plugins
        state: directory
        mode: 0775
        owner: ubuntu
        group: ubuntu

    - name: Descargar docker compose
      get_url:
        url: https://github.com/docker/compose/releases/download/v2.15.1/docker-compose-linux-x86_64
        dest: /home/ubuntu/.docker/cli-plugins/docker-compose
        mode: 0775
        owner: ubuntu
        group: ubuntu

    - name: Añadimos el usuario al grupo docker
      user:
        name: ubuntu
        groups: docker
        append: yes

    - name: Permisos de socket docker
      file:
        path: /var/run/docker.sock
        mode: 0777

    - name: Reiniciamos docker
      service:
        name: docker
        state: restarted

    - name: Copiamos el archivo
      copy:
        src: ../../docker/
        dest: /home/ubuntu/docker

    - name: Ejecutar docker-compose
      docker_compose:
        project_src: /home/ubuntu/docker
        files:
          - docker-compose.yml
        state: present
        docker_host: unix://var/run/docker.sock
```

## **Comprobación**


Si pongo el dominio en el navegador, aparecerá wordpress (ya lo habia instalado):

![](https://github.com/Daniel-Magana512/mkdocks/blob/main/fotos_practicas/foto_practica_14_1/foto1.png?raw=true)

Si pongo el dominio (poniendo http) y el puerto:8080 me llevará a phpmyadmin:
![](https://github.com/Daniel-Magana512/mkdocks/blob/main/fotos_practicas/foto_practica_14_1/foto2.png?raw=true)

