# Actividad 13.4 Ansible-Playground

*En esta actividad vamos a crear la arquitectura de la práctica 9 y 7.*

*Para generar la infraestructura, tenemos que estar dentro del directorio playbook y escribir **ansible-playbook "nombre_plantilla"**.El problema de hacerlo con ansible es que tarda mucho en generar las instancias, pero no tiene mucha dificultad.*

*Ahora voy a explicar el contenido de los archivos.*

## **Para la arquitectura de la práctica 7.**

Voy a explicar el archivo de creación de la arquitectura de la práctica:

```yml
---
- name: Este Playbook generá la arquitectura de la plantilla 7
  
  #La conexión va a ser de forma local, solamente para crear las instancias, de manera que no voy a tener necesidad de conectarme a ellas para ejecutar comandos por eso no hay un inventario y en la conexión se hace de forma local. 

  hosts: localhost
  connection: local
  gather_facts: false

  tasks:

  #Añadimos el archivo de las variables

  - name: Añadimos las variables
    ansible.builtin.include_vars:
      ../variables/variables.yml

    #Aquí empezamos a crear los grupos de seguridad de cada máquina.

    #Para la creación de los grupos de seguridad, vamos a usar el módulo "ec2_group", una vez declarado le pasaremos el nombre del grupo , la descripción y los roles(aquí hacemos la referencia a los puertos , indicamos elementos como el puerto que queremos abrir , las condiciones de la IP que debe de cumplir para interactuar con las máquinas, el tipo de puerto, y el protocolo )

  - name: Crear un grupo de seguridad del frontend
    ec2_group:
      name: "{{ ec2_security_group_frontend }}"
      description: "{{ ec2_security_group_description_frontend }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_http }}"
        to_port: "{{ port_http }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_https }}"
        to_port: "{{ port_https }}"
        cidr_ip: 0.0.0.0/0

    #Los mismos pasos para el backend

  - name: Crear un grupo de seguridad del backend
    ec2_group:
      name: "{{ ec2_security_group_backend }}"
      description: "{{ ec2_security_group_description_backend }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_mysql }}"
        to_port: "{{ port_mysql }}"
        cidr_ip: 0.0.0.0/0

#Aquí creamos las instancias.

#Para generar las instancias vamos a usar el módulo "ec2_instance", le pasamos los siguientes parametros: el nombre de la instancia, la clave ssh, el nombre del grupo , el tipo de instancia (capacidad), y la ami. Ponemos que el estado de la máquina sea running (que significa que esté funcionando).

#------------------------------------------------
#Creación del frontend
#-----------------------------------------------

#La última linea "register: ec2" es para que tener un registro de la máquina que luego usaremos para asociar la ip elástica, por eso la otra instancia no se tener esta línea.

  - name: Crear instancia del frontend
    ec2_instance:
      name: "{{ ec2_instance_name_frontend }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_frontend }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running
    register: ec2

#------------------------------------------------------
#Creación backend
#------------------------------------------------------
  - name: Crear instancia del backend
    ec2_instance:
      name: "{{ ec2_instance_name_backend }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_backend }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running

#No funciona la parte de asociar la ip elásticas al frontend
#-----------------------------------------------------------
#  - name: Mostramos el contenido de la variable ec2
#    debug:
#      msg: "ec2: {{ ec2 }}"
#
#  - name: Crear una nueva IP elástica y asociar a la instancia
#    ec2_eip:
#      device_id: "{{ ec2.instances[0].instance_id }}"
#    register: eip

#  - name: Mostrar la IP elástica
#    debug:
#      msg: "La IP elástica es: {{ eip.public_ip }}"
```

El archivo de las variables es el siguiente:
```yml
#Variables para crear el frontend, grupo de seguridad...
ec2_security_group_frontend: sg_frontend
ec2_security_group_description_frontend: Grupo de seguridad del frontend
ec2_instance_name_frontend: frontend

#Variables para crear el backend, grupo de seguridad...
ec2_security_group_backend: sg_backend
ec2_security_group_description_backend: Grupo de seguridad del backend
ec2_instance_name_backend: backend
#Variables puertos
port_ssh: 22
port_http: 80
port_https: 443
port_mysql: 3306

#Variables que van a tener todas las instancias
ec2_image: ami-06878d265978313ca
ec2_instance_type: t2.small
ec2_key_name: vockey
```

**Importante**, para destruir la infraestructura de la práctica7.

```yml
---
- name: Playbook para eliminar un grupo de seguridad y una instancia EC2 en AWS

  #La conexión va a ser de forma local, solamente para crear las instancias, de manera que no voy a tener necesidad de conectarme a ellas para ejecutar comandos por eso no hay un inventario y en la conexión se hace de forma local.

  hosts: localhost
  connection: local
  gather_facts: false


  tasks:

  #Añadimos las variables de los recursos de la infraestructura.

    - name: Añadimos las variables
      ansible.builtin.include_vars:
        ../variables/variables.yml

#Al no asociar la ip elástica porque no funciona , esta parte la comentaré.

    # - name: Obtener el id de instancia a partir del nombre
    #   ec2_instance_info:
    #     filters:
    #       "tag:Name": "{{ ec2_instance_name }}"
    #       "instance-state-name": "running"
    #   register: ec2

 #   - name: Desasociar de IP elástica de la instancia EC2
 #     ec2_eip:
 #       device_id: "{{ ec2.instances[0].instance_id }}"
 #       state: absent


#Ahora llegamos a la parte de eliminar las instancias.

#Para ello usaremos el módulo ec2_instance, le pasamos el nombre de la instancia y de estado pondremos absent (para indicarle que queremos borrarlo.)


    - name: Eliminar la instancia frontend
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_frontend }}"
        state: absent

    - name: Eliminar la instancia backend
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_backend }}"
        state: absent

#Ahora vamos a eliminar los grupos de seguridad, para ello usaremos el módulo ec2_group, le pasaremos el nombre del grupo y que tenga el estado absent.

    - name: Eliminar el grupo de seguridad frontend
      ec2_group:
        name: "{{ ec2_security_group_frontend }}"
        state: absent

    - name: Eliminar el grupo de seguridad backend
      ec2_group:
        name: "{{ ec2_security_group_backend }}"
        state: absent
```

## **Para la arquitectura de la práctica 9.**

Voy a explicar la plantilla9.yml, el contenido que tiene es el siguiente:

```yml
---
- name: Este Playbook generá la arquitectura de la plantilla 9

  #La conexión va a ser de forma local, solamente para crear las instancias, de manera que no voy a tener necesidad de conectarme a ellas para ejecutar comandos por eso no hay un inventario y en la conexión se hace de forma local.

  hosts: localhost
  connection: local
  gather_facts: false


  tasks:

  - name: Añadimos las variables
    ansible.builtin.include_vars:
      ../variables/variables.yml

    #Para la creación de los grupos de seguridad, vamos a usar el módulo "ec2_group", una vez declarado le pasaremos el nombre del grupo , la descripción y los roles(aquí hacemos la referencia a los puertos , indicamos elementos como el puerto que queremos abrir , las condiciones de la IP que debe de cumplir para interactuar con las máquinas, el tipo de puerto, y el protocolo )

  - name: Crear un grupo de seguridad del balanceador
    ec2_group:
      name: "{{ ec2_security_group_balancer }}"
      description: "{{ ec2_security_group_description_balancer }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_http }}"
        to_port: "{{ port_http }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_https }}"
        to_port: "{{ port_https }}"
        cidr_ip: 0.0.0.0/0


  - name: Crear un grupo de seguridad del frontend
    ec2_group:
      name: "{{ ec2_security_group_frontend }}"
      description: "{{ ec2_security_group_description_frontend }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_http }}"
        to_port: "{{ port_http }}"
        cidr_ip: 0.0.0.0/0
  
  - name: Crear un grupo de seguridad del nfs
    ec2_group:
      name: "{{ ec2_security_group_nfs }}"
      description: "{{ ec2_security_group_description_nfs }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_nfs }}"
        to_port: "{{ port_nfs }}"
        cidr_ip: 0.0.0.0/0


  - name: Crear un grupo de seguridad del backend
    ec2_group:
      name: "{{ ec2_security_group_backend }}"
      description: "{{ ec2_security_group_description_backend }}"
      rules:
      - proto: tcp
        from_port: "{{ port_ssh }}"
        to_port: "{{ port_ssh }}"
        cidr_ip: 0.0.0.0/0
      - proto: tcp
        from_port: "{{ port_mysql }}"
        to_port: "{{ port_mysql }}"
        cidr_ip: 0.0.0.0/0


#Para generar las instancias vamos a usar el módulo "ec2_instance", le pasamos los siguientes parametros: el nombre de la instancia, la clave ssh, el nombre del grupo , el tipo de instancia (capacidad), y la ami. Ponemos que el estado de la máquina sea running (que significa que esté funcionando).

#La última linea "register: ec2" es para que tener un registro de la máquina que luego usaremos para asociar la ip elástica, por eso la otra instancia no se tener esta línea.

  - name: Crear instancia del balanceador
    ec2_instance:
      name: "{{ ec2_instance_name_balancer }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_balancer }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running
    register: ec2


  - name: Crear instancia del frontend01
    ec2_instance:
      name: "{{ ec2_instance_name_frontend }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_frontend }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running


  - name: Crear instancia del frontend02
    ec2_instance:
      name: "{{ ec2_instance_name_frontend02 }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_frontend }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running


  - name: Crear instancia del nfs
    ec2_instance:
      name: "{{ ec2_instance_name_nfs }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_nfs }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running


  - name: Crear instancia del backend
    ec2_instance:
      name: "{{ ec2_instance_name_backend }}"
      key_name: "{{ ec2_key_name }}"
      security_group: "{{ ec2_security_group_backend }}"
      instance_type: "{{ ec2_instance_type }}"
      image_id: "{{ ec2_image }}"
      state: running


#No me deja asociar ip elástica al balanceador
#--------------------------------
#  - name: Mostramos el contenido de la variable ec2
#    debug:
#      msg: "ec2: {{ ec2 }}"

#  - name: Crear una nueva IP elástica y asociar a la instancia
#    ec2_eip:
#      device_id: "{{ ec2.instances[0].instance_id }}"
#    register: eip

#  - name: Mostrar la IP elástica
#    debug:
#      msg: "La IP elástica es: {{ eip.public_ip }}"

```

El archivo de las variables es el siguiente:

```yml
#Variables para crear el balanceador, grupo de seguridad...
ec2_security_group_balancer: sg_balanceador
ec2_security_group_description_balancer: Grupo de seguridad del balanceador
ec2_instance_name_balancer: balanceador

#Variables para crear el frontend, grupo de seguridad...
ec2_security_group_frontend: sg_frontend
ec2_security_group_description_frontend: Grupo de seguridad del frontend
ec2_instance_name_frontend: frontend01
ec2_instance_name_frontend02: frontend02

#Variables para crear el nfs, grupo de seguridad...
ec2_security_group_nfs: sg_nfs
ec2_security_group_description_nfs: Grupo de seguridad del nfs
ec2_instance_name_nfs: nfs
  
#Variables para crear el backend, grupo de seguridad...
ec2_security_group_backend: sg_backend
ec2_security_group_description_backend: Grupo de seguridad del backend
ec2_instance_name_backend: backend

#Variables que van a tener todas las instancias
ec2_image: ami-06878d265978313ca
ec2_instance_type: t2.small
ec2_key_name: vockey

#Variables de puertos
port_ssh: 22
port_http: 80
port_https: 443
port_nfs: 2049
port_mysql: 3306
```

Para eliminar las instancias usaremos el archivo llamado "plantilla9_eliminar.yml" su contenido es el siguiente:

```yml
---
- name: Playbook para eliminar un grupo de seguridad y una instancia EC2 en AWS

   #La conexión va a ser de forma local, solamente para crear las instancias, de manera que no voy a tener necesidad de conectarme a ellas para ejecutar comandos por eso no hay un inventario y en la conexión se hace de forma local.

  hosts: localhost
  connection: local
  gather_facts: false


  tasks:

    - name: Añadimos las variables
      ansible.builtin.include_vars:
        ../variables/variables.yml

#Este paso no va a funcionar, porque no me deja asociar la ip elástica.

    # - name: Obtener el id de instancia a partir del nombre
    #   ec2_instance_info:
    #     filters:
    #       "tag:Name": "{{ ec2_instance_name }}"
    #       "instance-state-name": "running"
    #   register: ec2

 #   - name: Desasociar de IP elástica de la instancia EC2
 #     ec2_eip:
 #       device_id: "{{ ec2.instances[0].instance_id }}"
 #       state: absent


#Ahora llegamos a la parte de eliminar las instancias.

#Para ello usaremos el módulo ec2_instance, le pasamos el nombre de la instancia y de estado pondremos absent (para indicarle que queremos borrarlo.)

    - name: Eliminar la instancia balanceador
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_balancer }}"
        state: absent

    - name: Eliminar la instancia frontend 01
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_frontend }}"
        state: absent

    - name: Eliminar la instancia frontend02
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_frontend02 }}"
        state: absent

    - name: Eliminar la instancia backend
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_backend }}"
        state: absent


    - name: Eliminar la instancia nfs
      ec2_instance:
        filters:
          "tag:Name": "{{ ec2_instance_name_nfs }}"
        state: absent

#Ahora vamos a eliminar los grupos de seguridad, para ello usaremos el módulo ec2_group, le pasaremos el nombre del grupo y que tenga el estado absent.

    - name: Eliminar el grupo de seguridad del balanceador
      ec2_group:
        name: "{{ ec2_security_group_balancer }}"
        state: absent

    - name: Eliminar el grupo de seguridad frontend01
      ec2_group:
        name: "{{ ec2_security_group_frontend }}"
        state: absent

    - name: Eliminar el grupo de seguridad frontend02
      ec2_group:
        name: "{{ ec2_security_group_frontend }}"
        state: absent

    - name: Eliminar el grupo de seguridad backend
      ec2_group:
        name: "{{ ec2_security_group_backend }}"
        state: absent
  
    - name: Eliminar el grupo de seguridad nfs
      ec2_group:
        name: "{{ ec2_security_group_nfs }}"
        state: absent

