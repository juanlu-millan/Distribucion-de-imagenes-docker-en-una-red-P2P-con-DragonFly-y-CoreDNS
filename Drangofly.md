![image](https://user-images.githubusercontent.com/43776895/119236256-acffb780-bb36-11eb-86d7-fc8bc335fdc4.png)

# Índice
- [Introducción](#introducción)
- [Prerequisitos](#prerequisitos)
- [Instalación](#instalación)
- [Verificación](#verificación)

## Introducción

Dragonfly es un sistema inteligente de distribución de archivos e imágenes basado en P2P de código abierto. Su objetivo es abordar todos los problemas de distribución en escenarios nativos de la nube. 

Las caracteristicas principales de Dragonfly :

* Simple : API orientada al usuario (HTTP) bien definida, no invasiva para todos los motores de contenedores.
* Eficiente : Compatibilidad con CDN, distribución de archivos basada en P2P para ahorrar ancho de banda empresarial.
* Inteligente : Límite de velocidad a nivel de host, control de flujo inteligente debido a la detección de host.
* Seguro : Cifrado de transmisión en bloque, soporte de conexión HTTPS.

- Dragonfly ahora está alojado en Cloud Native Computing Foundation (CNCF) como un proyecto de nivel de incubación. Originalmente nació para resolver todo tipo de distribución a escalas muy grandes, como distribución de aplicaciones, distribución de caché, distribución de registros, distribución de imágenes, etc.

- Dragonfly ha terminado de refactorizarse en Golang. Ahora las versiones> 0.4.0 están totalmente en Golang, mientras que las <0.4.0 están en Java. Recomendamos a los usuarios que prueben la versión de Golang primero, ya que las versiones de Java dejarán de ser compatibles en las próximas versiones.

## Prerequisitos

Suponiendo que el experimento de inicio rápido requiere que preparemos 4 máquinas host, una para desempeñar un papel de supernode y las otras 3 para dfclient. 

Veamos como seria la topología del clúster:

![image](https://user-images.githubusercontent.com/43776895/121814797-38222800-cc73-11eb-9be9-c99d75efbf88.png)

Entonces, debemos asegurarnos de los siguientes requisitos:

- 4 nodos de host en una LAN
- Cada nodo ha implementado dragonfly desta manera:

**Server**:  Dragonfly Server

**Host1**: Cliente Dragonfly

**Host2**: Cliente Dragonfly

**Host3**: Cliente Dragonfly

## Instalación

#### Paso 1: Implementar Dragonfly Server (Supernodo o Server)
Lanzamos en Docker a dragonfly indicando que sera el supernodo, utilizaremos los puerto 8001 y 8002.

<pre>
docker run -d --name supernode \
  --restart=always \
  -p 8001:8001 \
  -p 8002:8002 \
  -v /home/admin/supernode:/home/admin/supernode \
  dragonflyoss/supernode:1.0.2 --download-port=8001
</pre>

Comprobamos que ya esta en funcionamiento:

<pre>
CONTAINER ID   IMAGE                          COMMAND                  CREATED        STATUS             PORTS                                                           NAMES
1a79dfb2fa27   dragonflyoss/supernode:1.0.2   "/root/start.sh --do…"   44 hours ago   Up About an hour   0.0.0.0:8001-8002->8001-8002/tcp, :::8001-8002->8001-8002/tcp   supernode
</pre>

#### Paso 2: Implementar el cliente Dragonfly.

Las siguientes operaciones deben llevarse a cabo tanto en la máquina cliente dfclient0, dfclient1.

El archivo de configuración de Dragonfly se encuentra en el /etc/dragonfly que seria el directorio por defecto. Cuando utilice el contenedor para implementar el cliente, debe montar el archivo de configuración en el contenedor.

Configure la dirección de Dragonfly Supernode para el cliente:

<pre>
cat <<EOD > /etc/dragonfly/dfget.yml
nodes:
    - server.example.com
EOD
</pre>

Al utilizar un servidor DNS(CoreDNS) podemos asigar el nombre de la maquina.

Iniciar el cliente Dragonfly:

<pre>
docker run -d --name dfclient \
    --restart=always \
    -p 65001:65001 \
    -v /etc/dragonfly:/etc/dragonfly \
    -v $HOME/.small-dragonfly:/root/.small-dragonfly \
    dragonflyoss/dfclient:1.0.2 --registry https://index.docker.io
</pre>

Comprobamos que ya esta en funcionamiento:

<pre>
CONTAINER ID   IMAGE                         COMMAND                  CREATED        STATUS             PORTS                                           NAMES
9373f88d408e   dragonflyoss/dfclient:1.0.2   "/opt/dragonfly/df-c…"   45 hours ago   Up About an hour   0.0.0.0:65001->65001/tcp, :::65001->65001/tcp   dfclient
</pre>

#### Paso 3. Configurar el demonio de Docker

Tenemos que modificar la configuración  del Docker Daemon para utilizar dragonfly en todos los host que tengamos.

Agregue o actualice el elemento de configuración registry-mirrors en el archivo de configuración /etc/docker/daemon.json.

<pre>
{
  "registry-mirrors": ["http://127.0.0.1:65001"]
}
</pre>

Reiniciamos Docker .

<pre>
systemctl restart docker
</pre>

#### Paso 4: extraer imágenes con Dragonfly

A través de los pasos anteriores, podemos comenzar a validar si Dragonfly funciona como se esperaba realizando la siguiente prueba:

<pre>
docker pull nginx:latest
</pre>

Una vez realizado en uno de los equipos realizaremos el mismo procedimiento en otro host para comprobar si funciona.

## Verificación

#### Paso 5: validar Dragonfly
Puede ejecutar el siguiente comando para verificar si la imagen nginx se distribuye a través de Dragonfly.

<pre>
docker exec dfclient grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log
</pre>

Si la salida del comando anterior tiene contenido como la siguiente significa que la descarga de la imagen la realiza Dragonfly.

<pre>
2021-06-05 20:18:51.288 INFO sign:140-1622924331.218 : downloading piece:{"taskID":"05f945e758a52439048ab935efd0dfa49ca6963eaf5adb41883074aa5b435385","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:51.850 INFO sign:139-1622924331.215 : downloading piece:{"taskID":"40b4a6ba49a045c9082ea4d376ce85bdd9b89004a780fa129a85ba756842daee","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:51.866 INFO sign:138-1622924331.219 : downloading piece:{"taskID":"2d9d46e6f276f863ac1bd9657fb3838dc1e5d2c16d998b69fca7a3c6f7abd8f2","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:52.093 INFO sign:137-1622924331.216 : downloading piece:{"taskID":"9447f0fbd61cef2a75ee2f689fecab6f74d7c1348c747cce84b0171ff0555719","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:53.346 INFO sign:174-1622924333.340 : downloading piece:{"taskID":"e30cbaa7d78baf26a550ce5844f53870373d70ec5cc559885af7a3b1b2ab7865","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:53.406 INFO sign:180-1622924333.401 : downloading piece:{"taskID":"a5444918bcc613a274bcd1b0be69b273408a92f83176e8bef7871002673f0bdd","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:54.222 INFO sign:189-1622924334.209 : downloading piece:{"taskID":"d57c08ca46fcd9c223799a4fc7bc9dd3cdcfe3bd0060df95ba20260383c63226","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:18:54.558 INFO sign:197-1622924334.551 : downloading piece:{"taskID":"7b5674b1d185de889fa24aff2511a81b0a6b4a5fd63b5b4be84d31658cf53587","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:19:00.307 INFO sign:205-1622924340.297 : downloading piece:{"taskID":"1f0f6c994921ccd2c2adba82672c20c0d5158d39b4fe9e60e67e17cbecdedbc4","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
2021-06-05 20:19:00.498 INFO sign:212-1622924340.489 : downloading piece:{"taskID":"408bbb57f6a31bd04ce60cfea3fdea8acf0f06fa69cc2a23436925e1a2abc489","superNode":"server.example.com:8002","dstCid":"","range":"","result":502,"status":700,"pieceSize":0,"pieceNum":0}
</pre>

Si necesita asegurarse de que si la imagen se transfiere a través de otros nodos pares, puede ejecutar el siguiente comando:

<pre>
docker exec dfclient grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log | grep -v cdnnode
</pre>

Si el comando anterior no genera el resultado, el mirror no completa la transmisión a través de otros nodos pares. De lo contrario la transmisión se completa a través de otros nodos pares.
