![CoreDNS](https://cncf-branding.netlify.app/img/projects/coredns/horizontal/color/coredns-horizontal-color.png)

# Índice

- [Introducción](#introducción)
- [Funcionamiento en Docker](#funcionamiento-en-docker)
- [Ficheros de configuración](#ficheros-de-configuración)
- [Problemas a tener en cuenta para su funcionamiento en Docker](#problemas-a-tener-en-cuenta-para-su-funcionamiento-en-Docker)

## Introducción

¿Qué es CoreDNS?

CoreDNS es un servidor DNS y que particularmente está escrito en Go.

CoreDNS es diferente de otros servidores DNS, como pueden ser BIND,Knot o PowerDNS y Unbound ya que es muy flexible y casi todas las funciones se pueden ir agregando en complementos.

Los complementos pueden ser independientes o trabajar juntos para realizar una "función DNS".

Entonces, ¿qué es una "función DNS"? Para el propósito de CoreDNS, lo definimos como una pieza de software que implementa la API del complemento CoreDNS. 
La funcionalidad implementada puede desviarse enormemente. Hay complementos que no crean por sí mismos una respuesta, como métricas o caché, pero que agregan 
funcionalidad. Luego están los plugins que no generan una respuesta. Estos también pueden hacer cualquier cosa: hay complementos que se comunican con Kubernetes 
para proporcionar descubrimiento de servicios, complementos que leen datos de un archivo o una base de datos.

Actualmente hay alrededor de 30 complementos incluidos en la instalación predeterminada de CoreDNS, pero también hay un montón de complementos externos que 
puede compilar en CoreDNS para ampliar su funcionalidad.

CoreDNS fuçnciona con complementos.

Escribir nuevos complementos debería ser bastante fácil, pero requiere conocer Go y tener una idea de cómo funciona el DNS. CoreDNS abstrae muchos detalles 
de DNS, por lo que puede concentrarse en escribir la funcionalidad del complemento que necesita.

## Funcionamiento en Docker

Primero, extraemos la imagen del contenedor localmente desde Docker Hub:

<pre>
docker pull coredns/coredns
</pre>

Para ejecutar el contenedor CoreDNS busca en el directorio inmediato en el que se encuentra cualquier archivo con nombre Corefile y lo usa como configuración. 

Desafortunadamente, en la coredns/corednsimagen que extrajimos de Docker Hub, se encuentra en el directorio raíz de /, que no se puede montar como un volumen, pro eso tendremos que pasar manualmente nuestro Corefile y asegurarnos de que la directiva en nuestra zona de example.com:53 sea una ruta directa en el contenedor al archivo de base de datos de la zona DNS. Para hacer esto, le asigne a /root la ruta con la opción -conf que permite al usuario especificar la ruta a un Corefile.

<pre>
docker run -d --name coredns --restart=always --volume=/home/vagrant/containers/coredns/:/root/ -p 53:53/udp coredns/coredns -conf /root/Corefile
</pre>


## Ficheros de configuración

- Los principales ficheros de configuración son Corefile donde se definen las zonas de CoreDNS y Example.db:

##### Corefile

<pre>
.:53 {
    forward . 8.8.8.8 9.9.9.9
    log
    errors
    reload 10s
}

example.com:53 {
    file /root/example.db
    log
    errors
    erratic {
        delay 3 50ms
    }
    health :8080
    whoami
}
</pre>

Repasemos las opciones del Corefileuno por uno. Es importante tener en cuenta que cada sección entre corchetes denota una "zona" de DNS, que establece el comportamiento de CoreDNS en función de lo que se está resolviendo.

Primero, observe la sección inicial entre corchetes. Comienza con a .:53, lo que indica que esta zona es global (con "." Indica todo el tráfico), y está escuchando en el puerto 53 (udp por defecto). Los parámetros que establezcamos aquí se aplicarán a todas las consultas DNS entrantes que no especifiquen una zona específica, como una consulta para resolver github.com. Vemos en la siguiente línea, que reenviamos tales solicitudes a un servidor DNS secundario para su resolución; en este caso, todas las solicitudes a esta zona simplemente se reenviarán a los servidores DNS de Google en 8.8.8.8y 9.9.9.9.

En segundo lugar, tenemos otra zona que está especificada para que example.comtambién escuche en el puerto UDP 53. Cualquier consulta para los hosts que pertenecen a esta zona se referirá a una base de datos de archivos (similar a cómo lo hace el enlace) para realizar una búsqueda allí; más sobre eso momentáneamente. Por ejemplo, una consulta a "server.example.com" omitirá la zona global de "." y caiga en la zona que está dando servicio a "example.com", y utilizando la filedirectiva se hará referencia al archivo de la base de datos para encontrar el registro adecuado.

##### Example.db

<pre>
example.com.        IN  SOA dns.example.com. juanlu.example.com. 2021052512 7200 3600 1209600 >
gateway.example.com.    IN  A   192.168.121.1
dns.example.com.    IN  A   192.168.121.139
host1.example.com.   IN  A   192.168.121.245
host2.example.com.   IN  A   192.168.121.165
host3.example.com.   IN  A   192.168.121.57
jmillan.example.com.  IN  A   192.168.121.100
server.example.com. IN  CNAME dns
</pre>

Glosario de Example.db: 

- **example.com.** se refiere a la zona de la que es responsable este servidor DNS.
SOAse refiere al tipo de registro; en este caso, un "inicio de autoridad"

- **dns.example.com.** se refiere al nombre de este servidor DNS

- **juanlu.example.com.** se refiere al correo electrónico del administrador de este servidor DNS. Tenga en cuenta que el @signo simplemente se anota con un punto; esto no es un error, sino cómo está formateado.

- **2021052512** se refiere al número de serie. Puede ser el que desee, siempre que sea un número de serie que no se reutilice en esta configuración o que tenga caracteres no válidos.

- **7200** se refiere a la frecuencia de actualización en segundos; después de este período de tiempo, el cliente debe volver a recuperar una SOA.

- **3600** es la tasa de reintentos en segundos; después de esto, se debe reintentar cualquier actualización que haya fallado.

- **1209600** se refiere a la cantidad de tiempo en segundos que pasa antes de que un cliente ya no considere esta zona como "autorizada". La información de esta SOA caduca después de este tiempo.

- **3600** se refiere al tiempo de vida en segundos, que es el valor predeterminado para todos los registros de la zona.

### Pruebas de funcionamiento

**server.example.com**

<pre>
vagrant@host1:~$ dig server.example.com

; <<>> DiG 9.16.1-Ubuntu <<>> server.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38821
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: 6904e9e70440b521 (echoed)
;; QUESTION SECTION:
;server.example.com.		IN	A

;; ANSWER SECTION:
server.example.com.	0	IN	A	192.168.121.139

;; Query time: 0 msec
;; SERVER: 192.168.121.139#53(192.168.121.139)
;; WHEN: Mon Jun 07 21:36:07 UTC 2021
;; MSG SIZE  rcvd: 93
</pre>

**host1.example.com**

<pre>
vagrant@host1:~$ dig host1.example.com

; <<>> DiG 9.16.1-Ubuntu <<>> host1.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31117
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: b227d827554e247d (echoed)
;; QUESTION SECTION:
;host1.example.com.		IN	A

;; ANSWER SECTION:
host1.example.com.	0	IN	A	192.168.121.245

;; Query time: 0 msec
;; SERVER: 192.168.121.139#53(192.168.121.139)
;; WHEN: Mon Jun 07 21:36:17 UTC 2021
;; MSG SIZE  rcvd: 91
</pre>

**dns.example.com**

<pre>
vagrant@host1:~$ dig dns.example.com

; <<>> DiG 9.16.1-Ubuntu <<>> dns.example.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3671
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: a5c5770d323859a0 (echoed)
;; QUESTION SECTION:
;dns.example.com.		IN	A

;; ANSWER SECTION:
dns.example.com.	0	IN	CNAME	server.example.com.
server.example.com.	0	IN	A	192.168.121.139

;; Query time: 3 msec
;; SERVER: 192.168.121.139#53(192.168.121.139)
;; WHEN: Mon Jun 07 21:36:25 UTC 2021
;; MSG SIZE  rcvd: 137
</pre>

### Problemas a tener en cuenta para su funcionamiento en Docker

- En caso de encontrarte este error al desplegar CoreDNS en Docker:

<pre>
Error response from daemon: Cannot restart container cd815fad7222: driver failed programming external connectivity on endpoint coredns (59709c3d04d76b52a454876568cc9c081f9ed6083492716715668f6898736a9a): Error starting userland proxy: listen udp4 0.0.0.0:53: bind: address already in use
</pre>

- Se debera cambiar en /etc/systemd/resolved.conf la linea de DNSStubListener y cambiarla a *No*

<pre>
DNSStubListener=no
</pre>

- Comprobamos que funciona

<pre>
docker ps -a
CONTAINER ID   IMAGE                          COMMAND                  CREATED       STATUS         PORTS                                                           NAMES
cd815fad7222   coredns/coredns                "/coredns -conf /roo…"   6 days ago    Up 5 seconds   53/tcp, 0.0.0.0:53->53/udp, :::53->53/udp                       coredns
fa29c7a05185   dragonflyoss/supernode:1.0.2   "/root/start.sh --do…"   13 days ago   Up 2 days      0.0.0.0:8001-8002->8001-8002/tcp, :::8001-8002->8001-8002/tcp   supernode
</pre>
