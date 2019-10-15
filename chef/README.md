# Chef plugin

## Instalación de Chef DK
Descargamos el siguiente paquete
```
$ cd Descargas
$ wget https://packages.chef.io/files/stable/chefdk/4.4.27/ubuntu/18.04/chefdk_4.4.27-1_amd64.deb
$ sudo dpkg -i chefdk_4.4.27-1_amd64.deb
Seleccionando el paquete chefdk previamente no seleccionado.
(Leyendo la base de datos ... 267466 ficheros o directorios instalados actualmente.)
Preparando para desempaquetar chefdk_4.4.27-1_amd64.deb ...
Desempaquetando chefdk (4.4.27-1) ...
Configurando chefdk (4.4.27-1) ...
Thank you for installing ChefDK!
You can find some tips on getting started at https://learn.chef.io
```
Comprobamos la instalación ejecutando
```
$ chef
...
```

## Creación de *recipes*
En el directorio `chef-repo` crearmos nuestros bloques con los comandos de `chef`.

### Files
Creamos el primer fichero de nombre hello.rb. Chef está escrito en ruby, nuestro primer fichero tendrá el siguiente contenido.
```
file 'fichero-test.txt' do
    content 'hello world'
end
```
La forma de aplicarlo sería:
```
$ chef-apply hello.rb
Recipe: (chef-apply cookbook)::(chef-apply recipe)
  * file[fichero-test.txt] action create
    - create new file fichero-test.txt
    - update content in file fichero-test.txt from none to b94d27
    --- fichero-test.txt        2019-10-15 11:04:22.283784014 +0200
    +++ ./.chef-fichero-test20191015-21094-dgnsno.txt   2019-10-15 11:04:22.283784014 +0200
    @@ -1 +1,2 @@
    +hello world
```
Esto creará el fichero `fichero-test.txt` en el directorio en el que hemos ejecutado el comando. Otro ejemplo en el que vamos a cambiar la acción `content` por `delete`.
```
file 'fichero-test.txt' do
    action :delete
end
```
### Packages & Services
Para instalar un package creamos el siguiente contenido en un nuevo fichero ruby:
```
package 'apache2' do
    action :install
end

service 'apache2' do
    action [:enable, :start]
end

file '/var/www/html/index.html' do
    content '<h1>Hello world!</h1>'
end
```
