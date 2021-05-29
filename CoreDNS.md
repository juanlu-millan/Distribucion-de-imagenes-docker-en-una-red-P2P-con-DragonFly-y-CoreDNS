![CoreDNS](https://cncf-branding.netlify.app/img/projects/coredns/horizontal/color/coredns-horizontal-color.png)

- [Introducción](#introduccion)

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


#### Problemas a tener en cuenta para su funcionamiento en Docker

En caso de encontrarte este error al desplegar CoreDNS en Docker:

<pre>
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
</pre>
