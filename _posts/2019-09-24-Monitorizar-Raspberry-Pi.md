---
layout: post
title: Monitorizar Raspberry Pi
categories: [raspberry pi, monitorización]
tags: [docker, raspberry pi, linux, raspbian, monitorización, telegraf, influxdb, grafana]
excerpt_separator: <!--more-->
url_influxdb: https://www.influxdata.com/get-influxdb/
url_telegraf: https://influxdata.staging.wpengine.com/time-series-platform/telegraf/
url_chronograf: https://www.influxdata.com/time-series-platform/chronograf/
url_grafana: https://grafana.com/
url_grafana_dashboard: https://grafana.com/grafana/dashboards/10578
published: true
---

No pude evitarlo... Los _dashboards_ y la monitorización es algo que siempre me ha llamado la atención, sobre todo cuando es útil 
para ver de un vistazo el estado de un sistema o de una aplicación. Y desde que conocí **Grafana** estaba claro que en algún momento 
acabaría implementando algún sistema en mi _Raspberry_ que la usara. Pero primero empecemos por partes, y en orden...

<!--more-->

En esta entrada veremos cómo configurar:

- [**Telegraf**]({{ page.url_telegraf }})
- [**InfluxDB**]({{ page.url_influxdb }})
- [**Grafana**]({{ page.url_grafana }})

> **IMPORTANTE**: En este tutorial se contempla la instalación en una _Raspberry Pi Model B_ con _Debian 9 Stretch_. Comprueba la 
> versión de tu sistema y modifica los _script_ en los casos que sea necesario.

# Telegraf

**Telegraf** es un agente encargado de recopilar datos o métricas de un determinado sistema y almacenarlos donde le indiquemos. 
En nuestro caso, va a almacenar datos como el uso de memoria, CPU, temperatura de los procesadores... y los enviará a _InfluxDB_, 
la cual se encargará de almacenarlos. Ambas aplicaicones pertenecen a _InfluxData_, por lo que se integran perfectamente.

## Instalación de Telegraf

El primer paso que vamos a realizar es añadir las claves PGP del repositorio de InfluxData a nuestro sistema, para 
integrarlo con el sistema de gestión de paquetes APT de Debian y mantenerlo actualizado:

```bash
$ curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -
$ echo "deb https://repos.influxdata.com/debian stretch stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

$ sudo apt-get update
$ sudo apt-get install telegraf
```

## Configuración de Telegraf

Una vez hecho esto, tendremos nuestro agente recopilador instalado en nuestra Raspberry... ¡pero tenemos que configurarlo! 
Para ello, accedemos como _root_ al fichero `/etc/telegraf/telegraf.conf` y lo actualizamos para que coincida con la siguiente 
configuración:

```
[[outputs.influxdb]]
  urls = ["http://127.0.0.1:8086"]
  database = "telegraf"
  skip_database_creation = true
```

Lo que estamos haciendo ahí es indicarle a Telegraf que envíe las métricas que recopile a InfluxDB, que estará en la misma máquina 
corriendo en el puerto 8086 (el estándar de InfluxDB) y que haga uso de la base de datos _telegraf_ y que no cree dicha base de datos, 
ya que lo haremos nosotros manualmente.

A continuación, es recomendable que, si vamos a exponer nuestro InfluxDB al exterior, le añadamos seguridad, por lo que, de ser así, 
deberemos configurar los siguientes campos en nuestro `telegraf.conf` con el usuario y contraseña que definamos en la base de datos más adelante.
Por simplificar, podríamos decir que para la BD _telegraf_, el usuario será _telegraf_ y la contraseña _telegrafPass_:

```
[[outputs.influxdb]]
  ## HTTP Basic Auth
  username = "telegraf"
  password = "telegrafPass"
```

Así mismo, podríamos configurar la conexión HTTPS también, pero puesto que nuestro InfluxDB va a ser accesible únicamente en nuestra 
máquina, por ahora omitiremos este paso (y lo dejaremos para un posterior post de securización :) ).

A continuación, para que Telegraf nos envíe información también del estado de la CPU y la GPU, necesitamos hacer unos pasos adicionales. 
Primero, deberemos agregar el usuario al grupo de vídeo, tal que así:

```bash
$ sudo usermod -G video telegraf
```

Y crear un nuevo fichero de configuración en `/etc/telegraf/telegraf.d/raspberrypitemp.conf` con la siguiente información:

```
[[inputs.file]]
  files = ["/sys/class/thermal/thermal_zone0/temp"]
  name_override = "cpu_temperature"
  data_format = "value"
  data_type = "integer"

[[inputs.exec]]
  commands = ["/opt/vc/bin/vcgencmd measure_temp"]
  name_override = "gpu_temperature"
  data_format = "grok"
  grok_patterns = ["%{NUMBER:value:float}"]
```

Y con esto ya tenemos configurado nuestro Telegraf, con lo que sólo nos queda arrancar el servicio y habilitarlo para que arranque 
automáticamente cuando se (re)inicie el sistema, por lo que ejecutamos:

```bash
$ sudo systemctl start telegraf

$ sudo systemctl enable telegraf
```

# InfluxDB

Llegados a este punto, tenemos un agente que está recopilando información del sistema, pero que no es capaz de almacenarla y gestionarla 
por sí mismo, sino que la está enviando a una base de datos... que debemos instalar y configurar. Se trata de **InfluxDB**.

## Instalación de InfluxDB

Para instalar InfluxDB, como ya hemos configurado el repositorio de paquetes de InfluxData para instalar Telegraf, podemos realizar 
la instalación simplemente ejecutando:

```bash
$ sudo apt-get install influxdb
```

## Configuración de InfluxDB

En InfluxDB ya no hay una interfaz disponible para gestionar las bases de datos, por lo que el puerto 8083 ha quedado libre 
y todo lo referente a la sección `[admin]` del fichero de configuración es ignorado por la aplicación.

Antes de arrancar el servicio, accedemos como _root_ al fichero `/etc/influxdb/influxdb.conf` y modificamos la configuración 
para que cuadre con la siguiente:

```
[http]
  enabled = true
  bind-address = ":8086"
  auth-enabled = false
```

Una vez configurado nuestro InfluxDB, arrancamos el servicio y lo configuramos para que se inicie al arrancar el sistema:

```bash
$ sudo systemctl start influxdb
$ sudo systemctl enabled influxdb
```

Una vez que tenemos InfluxDB corriendo en nuestro sistema, crearemos nuestra base de datos y configuraremos un 
usuario y una contraseña para acceder a la misma. Podríamos hacerlo por consola, pero InfluxData nos proporciona una 
herramienta más visual, como es [**Chronograf**]({{ page.url_chronograf }}). La podemos bajar para cualquier sistema 
y nos bastará con ejecutarla para disponer de una interfaz web corriendo en el puerto 8888. En mi caso la he bajado 
desde un ordenador con Windows. 

Para ello, desde el menú de _Configuración_ crearemos una nueva conexión especificando el _host_ y puerto (8086) 
y como base de datos, pondremos `_internal`. Los campos de usuario y contraseña de momento los dejamos vacíos, ya que aún no los 
hemos creado. Una vez creada la conexión, en el apartado _InfluxDB admin_, en _Databases_ crearemos una nueva base de datos 
llamada _telegraf_ y le pondremos una duración de `7d`, para que no nos almacene el tiempo infinitamente, que no nos interesa.

Acto seguido, en _Users_ crearemos un usuario _admin_ con una contraseña y le daremos todos los permisos, y después un usuario 
_telegraf_ con la contraseña que dijimos (TelegrafPass) y le concederemos todos los permisos también.

Tras esto, en la configuración de InfluxDB cambiaremos el campo `auth-enabled` por `true` y reiniciaremos la aplicación, la cual 
lo hará ya con la autenticación habilitada por defecto.

Si ahora volvemos a Chronograf y editamos la conexión añadiendo el usuario y la contraseña, y añadimos en _Dashboard_ el de 
_System_, veremos cómo en la pestaña de _Dashboards_ tenemos ya información sobre nuestra Raspberry :).

¡Ya sólo nos resta visualizarla como es debido!

# Grafana

**Grafana** es una utilidad que nos permitirá visualizar de manera muy efectiva los datos que recojamos de distintas fuentes, 
como por ejemplo InfluxDB o Prometheus (del que hablaré en otra entrada).

## Instalación de Grafana

Para instalar Grafana en nuestra Raspberry, nos bastará con el comando:

```bash
$ sudo apt-get install grafana
```

Sin embargo, esto requiere de un montón de dependencias que quizá no queramos en nuestro sistema, como entornos de escritorio. 
Por eso, en mi caso vamos a usar Docker (se da por sentado que está instalado ya en la Raspberry).

Para ello, lo primero que haremos será crear un contenedor para que Grafana tenga dónde almacenar los _plugin_ y su base de datos, 
y así no perderlo en los reinicios. Para ello:

```bash
$ docker volume create grafana-storage
```

Tras ello, podemos arrancar nuestro contenedor de Grafana con la siguiente instrucción (en contraseña pondremos la que queramos tener de _login_):

```bash
$ sudo docker run \
  -d \
  -p 3000:3000 \
  --name=grafana \
  -v grafana-storage:/var/lib/grafana \
  -e "GF_INSTALL_PLUGINS=grafana-piechart-panel" \
  -e "GF_SECURITY_ADMIN_PASSWORD=admin" \
  grafana/grafana
```

Y si accedemos a la URL `http://<ip>:3000` veremos cómo nos sale el cuadro de inicio de sesión de Grafana.

## Configuración de Grafana

Una vez tenemos Grafana instalado, deberemos seguir 2 pasos: Configurar nuestra fuente de datos y configurar nuestro _dashboard_.

[EN DESARROLLO...]
