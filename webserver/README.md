# WebServer plugin

## Configuración del *container*
Tenemos que crear un contenedor del tipo **www.ApacheHttpdServer** y lo que tenemos que facilitar es:

* `Default Document Root`, que será el sitio donde se despliegue el contenido estático. Por ejemplo: `/var/www`
* `Configuration Fragment Directory`, que será el sitio en el que se ubiquen los ficheros de configuración. Por ejemplo: `/etc/apache2/sites-enabled`

## Configuración de los *deployables*

### www.WebContent
Aquí estará un zip con todo el contenido estático que queramos desplegar, por ejemplo:
```
└── documentRootDelCapi
    ├── img
    │   ├── img00.gif
    │   ├── img01.gif
    │   └── img02.gif
    ├── index.html
    └── static
        ├── page01.html
        └── page02.html
```
Este zip se descomprimirá en el directorio que hemos configurado en el container como `Default Document Root`, en nuestro caso, `/var/www`.

**El primer nivel de este zip debería ser un directorio que luego utilizaremos para definir el virutal host.**

### www.ApacheVirtualHostSpec
Aquí indicamos el nombre del host, el puerto, y el DocumentRoot. Por ejemplo:

* `Host` example.host.com
* `Port` 1111
* `Document Root` /var/www/documentRootDelCapi

Este último campo debe estar en consonancia con lo que hemos configurado en el container en el campo `Default Document Root`. Esto generará en el directorio `/etc/apache2/sites-enabled` un fichero con el siguiente contenido.
```
Listen 1111

<VirtualHost example.host.com:1111>
	DocumentRoot  /var/www/documentRootDelCapi
	ServerName example.host.com
</VirtualHost>
```
El nombre del fichero generado será el mismo que el nombre del *deployable* y se le añadirá la extensión '.conf'

### www.ApacheConfFile
Aquí indicamos un fichero de configuración que será ubicado en el directorio `/etc/apache2/sites-enabled`. Debe ser un fichero. Si se adjunta un zip no se extraerá y fallará el inicio de Apache.

**Importante:** el nombre del *deployable* será el nombre con el que quede en el directorio y **debe tener extensión .conf** para que lo lea Apache al iniciarse. Es decir, el nombre del *deployable* deberá ser algo como
```
001-apache.conf
```

### www.ApacheConfFragmentSpec
Se utiliza para generar contenido en base a templates ftl definidas previamente. Un ejemplo de ello es el *deployable* de tipo `www.ApacheVirtualHostSpec`.

Tendríamos que:

1. Definir el synthetic.xml
2. Crear la template ftl con el contenido a renderizar

En el sythetic.xml vemos lo siguiente:
```
<type type="www.ApacheVirtualHost" extends="www.ApacheConfFragment" deployable-type="www.ApacheVirtualHostSpec" description="Apache virtual host">
    <generate-deployable type="www.ApacheVirtualHostSpec" extends="www.ApacheConfFragmentSpec" description="Specification for an Apache virtual host"/>
    <property name="host" description="The virtual host name. Use '*' to match all hosts" />
    <property name="port" description="The virtual host port. Use '*' to match all ports" />
    <property name="documentRoot" required="false" description="The document root for ...." />
</type>
```
Vemos que extiende de `www.ApacheConfFragment`, éste a su vez extiende a `generic.ProcessedTemplate` que a su vez genera el deployable de tipo `www.ApacheConfFragmentSpec`

```
<type type="www.ApacheConfFragment" extends="generic.ProcessedTemplate" deployable-type="www.ApacheConfFragmentSpec" container-type="www.ApacheHttpdServer" description="Deployed generated Apache configuration file">
    <generate-deployable type="www.ApacheConfFragmentSpec" extends="generic.Resource" description="Specification for an Apache configuration fragment."/>
    <property name="createVerb" default="Deploy" hidden="true" />
    <property name="destroyVerb" default="Remove" hidden="true" />
    <property name="targetFile" default="${deployed.name}.conf" description="Target file name" hidden="true"/>
    <property name="targetDirectory" hidden="true" default="${deployed.container.configurationFragmentDirectory}" description="Target directory name" />
    <property name="template" hidden="true" default="www/apache/${deployed.type}.conf.ftl" description="Configuration fragment template file name."/>
    <property name="restartRequired" kind="boolean" required="false" default="true" hidden="true"/>
</type>
```
Vemos que las templates tienen que ubicarse en el directorio **www/apache** y deben tener el nombre del tipo de *deployable*. Si echamos un vistazo a este directorio veremos lo siguiente:
```
└── www
    └── apache
        ├── www.ApacheProxyPass.conf.ftl
        └── www.ApacheVirtualHost.conf.ftl
```
Esto son precisamente dos nuevos tipos de *deployables* que están disponibles. Si echamos un vistazo a la template del VirtualHost vemos lo siguiente:
```
Listen ${deployed.port}

<VirtualHost ${deployed.host}:${deployed.port}>
        DocumentRoot <#if deployed.documentRoot != ""> ${deployed.documentRoot}<#else> ${deployed.container.defaultDocumentRoot}${deployed.container.host.os.fileSeparator}${deployed.deployable.name}</#if>
        ServerName ${deployed.host}
</VirtualHost>
```
Que es justo el contenido que se genera cuando utilizamos un *deployable* de tipo VirtualHost.