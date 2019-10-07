https://blog.sonatype.com/using-nexus-3-as-your-repository-part-3-docker-images

El repositorio Docker requiere dos puertos para funcionar.
Vamos a usar el puerto 8083 para hacer pull desde el proxy y el puerto 8084 para hacer pull y push del repositorio privado.

Creación del repositorio privado.
Creamos un nuevo repositorio privado de tipo Docker (hosted) con las siguientes caracteristicas
name: docker-private
online: si
creamos acceso http, marcamos el check
Creamos el puerto 8084
Disable to allow anonymous acces: si
Enable Docker V1 API: si
Storage: deafult
Strict content type validation: si
Hosted: deployment policy: allow redeploy

Creamos un nuevo repositorio de tipo Docker Hub.
name: docker-hub
online: si
Disable to allow anonymous acces: si
Remote storage: ponemos la url del repositorio con el que queremos conectar: https://registry-1.docker.io/
Docker index: use docker hub.
Auto block enable: si
maximum component age: 1440
Maximum metadata age: 1440
Storage: default
Validate content: si
El resto de opciones "Negative cache" y "http" por defecto

Creamos también un nuevo repositorio de tipo DockerGroup para agrupar los dos anteriores para tener una única url con la que configurar los clientes.
name: docker-group
online: yes
Conectores
http: si
puerto: 8083
El resto de opciones por defecto y en el grupo añadimos los dos repositorioes que hemos creado antes.

configuramos ahora el cliente docker para usar el repositorio nexus. Para ello:
0) incluimos una nueva entrada en el fichero /etc/hosts con
lyhsoft-registry  127.0.0.1

1) Permitimos el acceso http (en lugar de https) incluyendo las siguiente líneas en el fichero /etc/docker/daemon.json
{
  "insecure-registries": [
    "lyhsoft-registry:8083",
    "lyhsoft-registry:8084"
  ],
  "disable-legacy-registry": true   # Esta linea la quitamos porque si no no arranca Docker
}

2) Reiniciamos docker con sudo systemctl restart docker

3) Ahora nos autenticamos con user y password en el nuevo repositorio
docker login -u admin -p admin123 lyhsoft-registry:8083
docker login -u admin -p admin123 lyhsoft-registry:8084

Esto creará el fichero ~/.docker/config.json

Hacer pull de imágenes. Para ello ejecutar
docker pull lyhsoft-registry:8083/httpd:2.4-alpine

Esto descargará la imagen y quedará en nexus en el repositorio docker-hub y docker-group

Construcción de una imagen
docker build --tag tutorial:4 .
Etiquetado
docker tag tutorial:4 lyhsoft-registry:8084/tutorial:4
Subida
docker push lyhsoft-registry:8084/tutorial:4
Descarga
docker pull lyhsoft-registry:8084/tutorial:4

Ejemplo:
