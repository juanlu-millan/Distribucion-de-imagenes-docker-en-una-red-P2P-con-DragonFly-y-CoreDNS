![image](https://user-images.githubusercontent.com/43776895/119236256-acffb780-bb36-11eb-86d7-fc8bc335fdc4.png)

# Índice
- [Introducción](#introducción)
- [Prerequisitos](#prerequisitos)
- [Instalación](#instalación)
- [Verificación](#verificación)

## Introducción

Dragonfly es un sistema inteligente de distribución de archivos e imágenes basado en P2P de código abierto. Su objetivo es abordar todos los problemas de distribución en escenarios nativos de la nube. 

Las caracteristicas principales de Dragonfly :

Simple : API orientada al usuario (HTTP) bien definida, no invasiva para todos los motores de contenedores;
Eficiente : compatibilidad con CDN, distribución de archivos basada en P2P para ahorrar ancho de banda empresarial;
Inteligente : límite de velocidad a nivel de host, control de flujo inteligente debido a la detección de host;
Seguro : cifrado de transmisión en bloque, soporte de conexión HTTPS.
Dragonfly ahora está alojado en Cloud Native Computing Foundation (CNCF) como un proyecto de nivel de incubación. Originalmente nació para resolver todo tipo de distribución a escalas muy grandes, como distribución de aplicaciones, distribución de caché, distribución de registros, distribución de imágenes, etc.

Dragonfly ha terminado de refactorizarse en Golang. Ahora las versiones> 0.4.0 están totalmente en Golang, mientras que las <0.4.0 están en Java. Recomendamos a los usuarios que prueben la versión de Golang primero, ya que las versiones de Java dejarán de ser compatibles en las próximas versiones.

## Prerequisitos

Suponiendo que el experimento de inicio rápido requiere que preparemos tres máquinas host, una para desempeñar un papel de supernodo y las otras dos para dfclient. Entonces, la topología del clúster de tres nodos es como la siguiente:

![Ejemplo](https://github.com/juanlu-millan/Distribucion-de-imagenes-docker-en-una-red-P2P-con-DragonFly-y-CoreDNS/blob/main/imagenes/ejemplo.png)

Entonces, debemos asegurarnos de los siguientes requisitos:

- 3 nodos de host en una LAN
- Cada nodo ha implementado el demonio de la ventana acoplable


## Instalación
Paso 1: Implementar Dragonfly Server (Supernodo)
Implemente el servidor Dragonfly (Supernode) en la máquina dfsupernode.

<pre>
docker run -d --name supernode \
  --restart=always \
  -p 8001:8001 \
  -p 8002:8002 \
  -v /home/admin/supernode:/home/admin/supernode \
  dragonflyoss/supernode:1.0.2 --download-port=8001
</pre>

Paso 2: Implementar el cliente Dragonfly (dfclient)
Las siguientes operaciones deben llevarse a cabo tanto en la máquina cliente dfclient0, dfclient1.

Prepare el archivo de configuración
El archivo de configuración de Dragonfly se encuentra en el /etc/dragonflydirectorio por defecto. Cuando utilice el contenedor para implementar el cliente, debe montar el archivo de configuración en el contenedor.

Configure la dirección de Dragonfly Supernode para el cliente:

<pre>
cat <<EOD > /etc/dragonfly/dfget.yml
nodes:
    - server.example.com
EOD
</pre>

Iniciar el cliente Dragonfly

<pre>
docker run -d --name dfclient \
    --restart=always \
    -p 65001:65001 \
    -v /etc/dragonfly:/etc/dragonfly \
    -v $HOME/.small-dragonfly:/root/.small-dragonfly \
    dragonflyoss/dfclient:1.0.2 --registry https://index.docker.io
</pre>


NOTA : El --registryparámetro especifica la dirección de registro de la imagen reflejada, y https://index.docker.ioes la dirección del registro de imagen oficial, también puede configurarlo para los demás.

Paso 3. Configurar el demonio de Docker
Tenemos que modificar la configuración del Docker Daemon utilizar la libélula como un tirón a través del registro tanto en la máquina cliente dfclient0, dfclient1.

Agregue o actualice el elemento de configuración registry-mirrorsen el archivo de configuración /etc/docker/daemon.json.

<pre>
{
  "registry-mirrors": ["http://127.0.0.1:65001"]
}
</pre>

Sugerencia: Para obtener más información sobre /etc/docker/daemon.json, consulte la documentación de Docker .

Reinicie Docker Daemon.

<pre>
systemctl restart docker
</pre>

Paso 4: extraer imágenes con Dragonfly

A través de los pasos anteriores, podemos comenzar a validar si Dragonfly funciona como se esperaba.
Y puede extraer la imagen como de costumbre en dfclient0o dfclient1, por ejemplo:

<pre>
docker pull nginx:latest
</pre>

## Verificación

Paso 5: validar Dragonfly
Puede ejecutar el siguiente comando para verificar si la imagen nginx se distribuye a través de Dragonfly.

<pre>
docker exec dfclient grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log
</pre>

Si la salida del comando anterior tiene contenido como


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

eso significa que la descarga de la imagen la realiza Dragonfly.

Si necesita asegurarse de que si la imagen se transfiere a través de otros nodos pares, puede ejecutar el siguiente comando:

<pre>
docker exec dfclient grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log | grep -v cdnnode
</pre>

Si el comando anterior no genera el resultado, el espejo no completa la transmisión a través de otros nodos pares. De lo contrario, la transmisión se completa a través de otros nodos pares.
