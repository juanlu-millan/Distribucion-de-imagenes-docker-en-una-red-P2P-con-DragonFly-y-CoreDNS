![CoreDNS](https://cncf-branding.netlify.app/img/projects/coredns/horizontal/color/coredns-horizontal-color.png)

# Índice

- [Introducción](#introduccion)
- [Funcionamiento en Docker](#funcionamiento-en-docker)
- [Ficheros de configuración](#ficheros-de-configuración)


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

<pre>
docker run -d --name coredns --restart=always --volume=/home/vagrant/containers/coredns/:/root/ -p 53:53/udp coredns/coredns -conf /root/Corefile
</pre>


## Ficheros de configuración

- Los principales ficheros de configuración son Corefile donde se definen las zonas de CoreDNS:

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

##### Example.db

<pre>
example.com.        IN  SOA dns.example.com. juanlu.example.com. 2015082542 7200 3600 1209600 >
gateway.example.com.    IN  A   192.168.121.1
dns.example.com.    IN  A   192.168.121.139
host1.example.com.   IN  A   192.168.121.245
host2.example.com.   IN  A   192.168.121.165
host3.example.com.   IN  A   192.168.121.57
jmillan.example.com.  IN  A   192.168.121.100
server.example.com. IN  CNAME   dns
</pre>


#### Problemas a tener en cuenta para su funcionamiento en Docker

En caso de encontrarte este error al desplegar CoreDNS en Docker:

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
