# Docker plugin

Para configurar el repositorio privado en Nexus3 se han seguido las siguientes intrucciones:

https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images

## Configuración de un repositorio Docker privado
El repositorio Docker requiere dos puertos para funcionar. Vamos a usar el puerto 8083 para hacer pull desde el proxy y el puerto 8084 para hacer pull y push del repositorio privado.

### Creación del repositorio privado.
Creamos un nuevo repositorio privado de tipo Docker (hosted) con las siguientes caracteristicas:

* name: docker-private
* online: si
* creamos acceso http, marcamos el check
* creamos el puerto 8084
* allow anonymous docker pull: no
* enable Docker V1 API: no
* storage: deafult
* strict content type validation: si
* hosted: deployment policy: allow redeploy
* cleanup policy: none

### Creación del repositorio proxy.
Creamos un nuevo repositorio de tipo Docker Hub (realmente no es necesario).

* name: docker-hub
* online: si
* disable to allow anonymous acces: si
* remote storage: ponemos la url del repositorio con el que queremos conectar: https://registry-1.docker.io/
* docker index: use docker hub.
* auto block enable: si
* maximum component age: 1440
* maximum metadata age: 1440
* storage: default
* validate content: si
* el resto de opciones "Negative cache" y "http" por defecto

### Creación de un grupo de repositorios.
Creamos también un nuevo repositorio de tipo DockerGroup para agrupar los dos anteriores para tener una única url con la que configurar los clientes (realmente no es necesario).

* name: docker-group
* online: yes
* Conectores
* http: si
* puerto: 8083
* el resto de opciones por defecto y en el grupo añadimos los dos repositorioes que hemos creado antes.

### Configuración del cliente docker.
Configuramos ahora el cliente docker para usar el repositorio nexus. Para ello:

1. incluimos una nueva entrada en el fichero /etc/hosts con `127.0.0.1 lyhsoft-registry`
2. Permitimos el acceso http (en lugar de https) incluyendo las siguiente líneas en el fichero `/etc/docker/daemon.json`
```
{
  "insecure-registries": [
    "lyhsoft-registry:8084"
  ]
}
```
3. Reiniciamos docker con sudo systemctl restart docker
4. Ahora nos autenticamos con user y password en el nuevo repositorio
```
$ docker login -u admin lyhsoft-registry:8084
```
5. Esto creará el fichero ~/.docker/config.json con el siguiente contenido
```
{
        "auths": {
                "https://index.docker.io/v1/": {
                        "auth": "bHloc29mdDoyMDAxamNsYQ=="
                },
                "lyhsoft-registry:8084": {
                        "auth": "YWRtaW46YWRtaW4xMjM="
                }
        },
        "HttpHeaders": {
                "User-Agent": "Docker-Client/19.03.2 (linux)"
        }
}
```

### Creación de un nuevo Docker Registry en XLD
```
# Creamos el registry privado en Nexus3 para Docker
dockerRegistryProperties = {
    'url': 'http://lyhsoft-registry:8084',
    'username': 'admin',
    'password': 'admin123'
    }
createResource("Configuration/lyhsoft-registry", "docker.Registry", dockerRegistryProperties)
```

## Hacer push al nuevo repositorio
Una vez que tengamos una imagen construída, podemos crear un tag que referencie al nuevo repositorio, por ejemplo:
```
# Construcción de una imagen
$ docker build --tag tutorial:4 .
# Etiquetado
$ docker tag tutorial:4 lyhsoft-registry:8084/tutorial:4
# Subida
$ docker push lyhsoft-registry:8084/tutorial:4
# Descarga
$ docker pull lyhsoft-registry:8084/tutorial:4
```

## Creación de los Docker Engine para diferentes entornos

### Creación de las máquinas

Vamos a utilizar la herramienta `docker-machine` para crear tres máquinas virtuales con el **Docker Engine** instalado. Además ejecutamos una serie scripts para permitir el acceso al Docker Registry local. Para ello:
```
echo "`cat <<EOF
{
  "insecure-registries": [
    "lyhsoft-registry:8084"
  ]
}
EOF`" > /tmp/daemon.json
for env in dev pre pro
do
  docker-machine create --driver virtualbox docker-${env}
  port=`docker-machine inspect --format='{{.Driver.SSHPort}}' docker-${env}`
  scp -o "StrictHostKeyChecking=no" -P${port} -i ~/.docker/machine/machines/docker-${env}/id_rsa /tmp/daemon.json docker@localhost:/tmp
  ssh -o "StrictHostKeyChecking=no" docker@localhost -p ${port} -i ~/.docker/machine/machines/docker-${env}/id_rsa 'sudo -- sh -c "mv /tmp/daemon.json /etc/docker/"'
  docker-machine restart docker-${env}
  ssh -o "StrictHostKeyChecking=no" docker@localhost -p ${port} -i ~/.docker/machine/machines/docker-${env}/id_rsa 'sudo -- sh -c "echo 10.0.2.2 lyhsoft-registry >> /etc/hosts"'
done
```

### Registro de las máquinas en XL Deploy
En el directorio `/home/jcla/.docker/machine/machines` habrá tres directorios, uno por cada máquina virtual. En cada uno de ellos estarán los certificados para acceder al docker engine. Con esta información creamos los correspondientes elementos de infraestructura en XL Deploy.
