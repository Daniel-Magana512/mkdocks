# Práctica 6 Prestashop mediante Ansible

Esta práctica es muy parecido a la que hice en el repositorio 5, la única diferencia es que vamos a utilizar una herramienta llamada _Ansible_ , esta herramienta, nos permite configurar los equipos de una red sin que tengamos que hacerlo uno a uno , esto hace que la configuración de las máquinas sea mucho más optimo.

Para ello vamos a utilizar archivos con la extensión *yml* (en estos archivos vamos a poner los comandos para llevar a cabo la administración de los equipos de nuestra red.

Cada vez que queramos ejecutar un archivo yml, deberemos de ejecutar el comando:

**ansible-playbook -i inventario install_lamp.yml**

Donde el archivo_lamp.yml es el que en mi caso he ejecutado, pero podría ser cualquiera

## **El archivo inventario**

Es importante ya que aquí definimos los grupos a con los que vamos a trabajar
Para ello habrá que especificar mediante las IPs de los los grupos de los equipos. (este no tendrá extensión)

* **Donde pone ansible_user especificamos el usuario con el que accedemos a esos equipos.**
* **ansible_ssh_private_key es la clave privada**
* **ansible_ssh_common_args es para que no nos pregunte si aceptamos los terminos al principio**

 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_1_6.PNG)


## **Archivo installl_lamp.yml**

Esto lo voy a explicar una vez ya que se repetirá cada vez que creemos un archivo con extensión yml con la herramienta de Ansible.


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_2_6.PNG)

Empezamos poniendo el --- , esto se utilizar para definir una lista.
**name** funciona solo de forma informativa.
Los **host** se utilizar para definir un grupo (que ya esta creada en el inventario).

**Become = yes**, es para decir que entramos como root.

La palabra **vars** es para decir que vamos a crear una variable , que tenga un lista que nosotros le pongamos, en este caso por ejemplo estaba diciendo que quiero los paquetes de php, más adelante declaro la variables php_paquetes y esos paquetes se deberán de instalar , sin tener que poner paquete por paquete para la instalación.

En el **tasks** vamos a ejecutar las operaciones.

* **La primera operación**  (actualizar repositorios) , pongo el name para saber que operación es. Después pongo apt por ser la libreria que útiliza este comando, update_cache es para que realizar la operación.


* **La segunda operación**  (Instalar el servidor web de apache) pongo el name primero que es para explicar lo que haré, el apt es por la librería y state es el estado en que me lo quiero instalar porque yo puedo instalarlo y que no este activo pues esas cosas se definen aquí.

* **Instalar PHP** funciona también con apt, en name vamos llamar a la variable que hemos creado con anterioridad (así los paquetes que hemos llamado antes, pues se descargarán e instalarán de golpe), el *state: present** le decimos que los queremos activos

* **Funcionalidades de phpmyadmin** Es lo mismo que proceso que en el anterior.

Pasamos a la siguiente foto de este archivo:


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_3_6.PNG)

* **Para borrar un archivo** debo de poner **file** para definir que es un archivo, **path** es la ubicación entera del archivo. Con el **state: absent**.

* **Para reiniciar un servicio** pongo **service** en name (nombre del servio) y en **state** (lo que busque) en mi caso pongo restarted para reinicar.


## **El archivo install_cerbot**

Aquí vamos a configurar todo lo necesario para poner utilizar el protocolo https , en nuestra tienda de **Prestashop** de manerá que nos comunicaremos con la tienda a través del puerto 443.


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_4_6.PNG)

Creo dos variables:

* **DOMAIN** que es donde meteré el dominio que he asociado la ip del cliente.

* **PS_EMAIL** es donde meteré el correo que quiera asociar con el dominio

Después ejecuto el siguiente contenido:

* **Lo primero de todo** me asegura que cerbot , no está instalado en la máquina.

* **Lo siguiente** es para instalar snap que es como la librería de apt, pero con otras funcionalidades.

* **Y por último** Descargo el certificado con las variables que he mencionado, para ello utilizo el **ansible.builtin.comand** 

* **Finalmente** Reiniciamos el servidor

## **Archivo installl_tools.yml**


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_5_6.PNG)

* La variable que pongo al principio viene dada por una operación que explicaré más adelante(básicamente cuando me descargue la herramienta que me dice las necesidades de prestashop me pedia esos módulos).

* **Unzip** Me permitira descomprimir archivos.

* **Primer Borrado y segundo borrado** es por si tengo la carpeta de prestashop en el directorio /var/www/html y deseo borrarlo para tener una instalación limpia y el segundo hace referencia a la herramienta de verificación de prestashop.

* **Clonación de la herramienta verificadora** Utilizo el git porque esa librería me permite clonar , **repo** es para poner la url y con  **path** pongo donde me gustaría tener los archivos.


* **Modificación de los permisos** Aquí pongo el directorio con **path** en **state: directry** digo que es un directorio. **owner** (pongo al usuario) y **group** (pongo al grupo) , porque el va a hacer referencia a usuarios de apache.

Seguimos con el archivo:


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_6_6.PNG)

* **Instalación de la variables que he generado antes** Una cosa que no he mencionado antes, es que las variables deben de estar dentro de {{Variable}} , así es como debería de estar.

* **Instalción de los modulos necesarios** pongo **apache2module** que me deja instalar los modulos necesarios de apache.

* **Cambiar el contenido del archivo php.ini**

Para ello pongo **ansible.bulting.lineinfile**

Con **Path** le digo la ubicación del archivo.

Con **search_string** Me busca el contenido que quiero sustituir.

Con **line** me permite poner la linea tal cual necesito, el problema de esto que me quita la linea completa, en la próxima práctica utilizaré **replace**.

Seguimos en el mismo archivo:

 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto__7_6.PNG)

* **Descargar un archivo** Esto equivale al comando wget, podria poner **get_url** y funcionaría. En la **url** pongo la dirección del archivo y en **dest** pongo el destino.

* **Descomprimir un archivo** Utilizamos **unarchive** que nos permitirá hacer la función mencionada. En **src** pongo la ubicación actual y en **dest** el destino.

* **Movemos los archivos de lugar y aprovechamos** para cambiarle los permisos , además ponemos recurse para que se haga de forma **recursiva**, es decir, para que lo haga en los subdirectorios.

Seguimos con el archivo:


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_8_6.PNG)

* **Añadimos los permisos de escritura necesario , además de los permisos de apache y loa hacemos de forma recursiva**

* **Descagamos PhpMyadmin** y la ubicamos donde queramos en mi caso /tmp

* **Descargamos la herramienta unzip para extraer archivos**

* **Eliminar el archivo.zip de phpmyadmin** no pasará nada porque ya hemos descomprimido la descarga.

* **Desplazmos el phpmyadmin** a /var/www/html , **importante** para mover el archivo de sitio debemos de ponerle **remote_src: yes** para que lo haga en el cliente, de lo contrario lo hará solo en el equipo local.

* **Añadimos los permisos de apache a PhpMyadmin y que sea de manera recursiva**

* **Reiniciamos el servidor apache**

## **Archivo deploy.yml**

En este archivo vamos a instalar y configurar **Prestashop** **PhpMyAdmin**


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_9_6.PNG)

Declaramos dos grupos de variables.

* **Primer grupo de variables** es para la base de datos execto **DOMAIN** y **PS_DB_SSL**

* **El segundo grupo de variables** es para los datos de usuario que vaya a entrar en la cuenta de la prágina web.

Nos vamos a los comandos

* **Python3-pip** es una librería como **apt** pero con otras funcionalidades, nos permitirá descargar **pymysql**.

* **pymsql** Nos es un modulo que nos permite comunicar la base de datos con la tienda.

* **Creación de la base de datos** usamos **mysql_db** para declara (acción) que vamos a crear una base de datos.En **name** ponemos la variable de la base de datos y **login_unix_socket** nos permite ejecutar el socket de mysql haciendola funcionar.

* **Creación del usuario en la base de datos** nos obliga a poner **no_log: true** , utilizaremos **mysql_user** por la acción de crear un usuario en la base de datos, en **name** ponemos el nombre de la base de datos. En **password** ponemos la contraseña de la base de datos, en **priv** añadimos los permisos sobre la base de datos , diciendole que tenga permisos sobre esta misma base de datos y de todas las tablas.  

Seguimos con el mismo archivo


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_10_6.PNG)

* **Vamos a Instalar Prestashop** con la opción de **ansible.builtin.command** nos permitirá insertar información (variables) al archivo **index_cli.php** la opción de *args* le pasamos el directorio donde se encuentra el archivo mencionado.

* **Después de la instalación nos obliga a borrar el archivo install por motivos de seguridad**

* **Movemos el archivo prestashop** a /var/www/html

* **Añadimos los permisos de prestashop** poniendo como propietario tanto grupo y usuario www-data lo hacemos de forma **recursiva**.

* **Reiniciamos el servidor Apache**

## **Archivo conf_apache.yml**

Cuando instalé todo en el cliente descargé los archivos **000-default.conf** y **000-default-le-ssl.conf** los metí en un repositorio de github de manerá que cuando quiera modificar algo solo tengo que modificarlo en github y hacer como **un git clone** borrando los iriginales y aplicanod las modificaciones


 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_11_6.PNG)

* **Eliminamos los archivos 000-default.conf y 000-default-le-ssl.conf**

* **Hago la clonación con los cambios efectuados**

* **Muevo los dos archivos al directorio /etc/apache2/sites-available/**

* **Reiniciamos el servidor de apache**



Esto es lo que he añadido en los dos archivos:

La primera línea es para que me coja por defecto el **prestashop**.

La segunda línea es para que me me **permita navegar** por la página web 

 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_12_6.PNG)
## **Comprobación de que funciona**

 ![](https://daniel-magana512.github.io/mkdocks/fotos_practicas/foto_practica_6/foto_13_6.PNG)