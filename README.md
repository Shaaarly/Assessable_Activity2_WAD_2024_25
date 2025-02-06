# Deploy con EC2

## 1. Cómo empezar

Primero que todo tenemos que entrar con nuestra cuenta de AWS al laboratorio para empezar el proceso. Una vez dentro, tenemos que iniciarlo. En nuestro caso, el laboratorio ya está encendido, por lo que aparece al lado de AWS un círculo verde. En caso de estar apagado, sale rojo.

Una vez está encendido, hay que darle donde AWS con el círculo verde para entrar en el panel.

---

## 2. Configurar grupos de seguridad

Lo primero que vamos a hacer dentro del panel es configurar las opciones de seguridad para habilitar la conexión SSH a través del puerto 22 y HTTP en el 80.

En este apartado le damos a crear grupo de seguridad y rellenamos los campos correspondientes.

Aquí, una vez buscado y seleccionado SSH, se nos pondrá automáticamente el protocolo TCP y el puerto 22. Lo único que tenemos que hacer es restringir el destino. Normalmente, se restringe solo a una IP por seguridad, pero en nuestro caso no importa, por lo que seleccionamos `0.0.0.0/0`. Lo mismo para HTML.

Con esto, ya tendríamos la configuración lista. Solo resta confirmar y crear el grupo. En nuestro panel podemos ver que lo tenemos creado correctamente.

---

## 3. Crear una instancia

Cuando tenemos todo esto configurado, vamos al apartado de instancias en el menú lateral izquierdo y seleccionamos la opción de "Lanzar una instancia".

Aquí de momento solo ponemos el nombre y seleccionamos nuestro sistema operativo, que será **Ubuntu**.

Justo debajo de la opción anterior tendremos para elegir la imagen de la máquina virtual. En nuestro caso elegimos **Ubuntu Server 22.04 LTS**.

Cuando se selecciona esta opción, aparece para elegir el tipo de procesador. Hemos elegido **ARM** por ser más eficiente en general.

Luego seleccionamos el tipo de instancia. Cada una tiene características y precios diferentes. En nuestro caso, elegimos **t4g.nano**.

A continuación, seleccionamos el par de claves que vamos a usar por seguridad. En nuestro caso, usamos el que ya viene predefinido en el laboratorio.

Por último, seleccionamos el grupo de seguridad creado anteriormente, configurado para protocolo TCP y conexión tanto para **SSH (puerto 22)** como para **HTTP (puerto 80)**.

Con esto ya tenemos la instancia configurada y podemos darle a "Lanzar instancia".

---

## 4. Configurar IP elástica

Una vez creada la instancia, podemos visualizarla en nuestro panel. Ahora necesitamos configurar una **IP elástica** porque cuando una instancia de AWS EC2 se reinicia, su IP pública cambia. Esto puede causar problemas de acceso y configuración.

Para evitarlo, asignamos una IP elástica, que es una dirección IP fija que permanece constante incluso si la instancia se apaga o reinicia. Esto garantiza estabilidad, facilita la configuración de dominios y evita errores en conexiones externas.

Para configurarlo:

1. Vamos al menú lateral izquierdo al apartado de "Seguridad".
2. Seleccionamos "Asignar dirección IP elástica".
3. No modificamos nada y damos a "Asignar".

En nuestro caso, se ha creado la IP `44.194.122.207`, que no está asignada aún. Solo queda asignarla:

1. Desde el menú de direcciones IP elásticas, seleccionamos la IP creada.
2. En el botón de acciones, elegimos "Dirección IP elástica asociada".
3. Seleccionamos la instancia creada anteriormente.
4. Damos a "Asignar".

Ahora podemos volver a nuestra instancia y veremos cómo han cambiado las IPs a las que hemos configurado recién.

---

## 5. Acceder a nuestra instancia

Ahora que tenemos todo configurado, debemos ir al panel principal de AWS para descargarnos nuestro archivo `.pem`.

Una vez descargado, usamos el siguiente comando para acceder a nuestra instancia con la **IPv4 pública**:

```sh
ssh -i labsuser.pem ubuntu@ec2-44-194-122-207.compute-1.amazonaws.com
```

La primera vez que intentamos ejecutar el `.pem` a través de SSH, nos da una alerta de que los permisos están demasiado abiertos. Para solucionarlo, hay que modificar los permisos del archivo:

```sh
chmod 400 labsuser.pem
```

Después de esto, ya podemos conectarnos sin problemas.

---

## 6. Configuración de la instancia

Una vez dentro de nuestra instancia, vamos a configurarla, ya que está vacía. Instalamos lo necesario:

### Actualizamos la lista de paquetes:

```sh
sudo apt update
```

### Instalamos Apache:

```sh
sudo apt install apache2 -y
```

### Instalamos npm:

```sh
sudo apt install npm -y
```

### Instalamos pm2:

```sh
npm install pm2 -g
```

Con esto, la instancia está lista para nuestro proyecto.

---

## 7. Configuración del proyecto

Ya con la instancia configurada, desde la terminal conectada a nuestra instancia, clonamos el proyecto desde el repositorio:

```sh
git clone [<URL-DEL-REPO>](https://github.com/gisgarme/adivina-el-numero)
```

Dentro del proyecto, editamos el `package.json` y añadimos esta línea:

```json
"homepage": "http://ec2-44-194-122-207.compute-1.amazonaws.com",
```

Luego instalamos las dependencias:

```sh
npm install
```

Y reiniciamos Apache para guardar los cambios:

```sh
sudo systemctl restart apache2
```

---

## 8. Configuración de Apache

Para finalizar, configuramos el servidor Apache. Editamos su archivo de configuración:

```sh
sudo nano /etc/apache2/sites-available/000-default.conf
```

En este archivo, indicamos tanto el dominio o la IP que Apache usará para identificar el VirtualHost:

```apache
ServerName ec2-44-194-122-207.compute-1.amazonaws.com
```

Comentamos `DocumentRoot`, ya que usaremos un **proxy inverso** en vez de un directorio estático:

```apache
# DocumentRoot /var/www/html
ProxyPass / http://localhost:5173/
ProxyPassReverse / http://localhost:5173/
```

El **ProxyPass** y **ProxyPassReverse** redirigen las solicitudes al puerto interno de Vite en vez del directorio base. Esto ayuda a ocultar el puerto interno, mejorando la seguridad.

Guardamos los cambios y activamos los módulos de Apache necesarios:

```sh
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo systemctl restart apache2
```

Ya casi lo tenemos, solo queda iniciar el proceso en segundo plano con pm2:

```sh
sudo pm2 start npm --name "vite-server" -- run dev -- --host
```

Ahora solo resta entrar con la IP configurada y visualizar la aplicación funcionando correctamente.
