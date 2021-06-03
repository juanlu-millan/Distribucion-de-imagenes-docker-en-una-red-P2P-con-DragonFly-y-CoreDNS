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
Paso 1: Implementar Supernodo (servidor Dragonfly)
Por encima de los tres nodos que preparamos, elegimos uno para implementar el supernodo.

Extraiga la imagen de la ventana acoplable que le proporcionamos.
docker pull dragonflyoss/supernode:0.3.1
Inicie un supernodo.
docker run -d -p 8001:8001 -p 8002:8002 dragonflyoss/supernode:0.3.1 -Dsupernode.advertiseIp=127.0.0.1
NOTA : supernode.advertiseIpdebe ser la ip a la que los clientes pueden conectarse, 127.0.0.1aquí hay un ejemplo para probar.

Paso 2. Configurar el demonio de Docker
Después de implementar Supernode en un nodo con éxito, deberíamos implementar dfclient (Dragonfly Client) en cada uno de los dos nodos restantes. Sin embargo, antes de implementar dfclient, debemos configurar Docker Daemon en ambos dos nodos para agregar el parámetro registry-mirrors.

Modifique el archivo de configuración /etc/docker/daemon.json.
vi /etc/docker/daemon.json
Sugerencia: Para obtener más información sobre /etc/docker/daemon.json, consulte la documentación de Docker .

Agregue o actualice el elemento de configuración registry-mirrorsen el archivo de configuración.
"registry-mirrors": ["http://127.0.0.1:65001"]
Reinicie Docker Daemon。
systemctl restart docker
Paso 3: Implementar dfclient (cliente Dragonfly)
Después de configurar el demonio docker de ambos nodos, podemos comenzar a implementar dfclient en ellos.

Extraiga dfclient en cada uno de los dos nodos:
docker pull dragonflyoss/dfclient:0.3.1
ejecute el comando en el primero de los dos nodos para iniciar dfclient
docker run -d --name dfclient01 -p 65001:65001 dragonflyoss/dfclient:0.3.1 --registry https://index.docker.io
ejecute el comando en el segundo de los dos nodos para iniciar dfclient
docker run -d --name dfclient02 -p 65001:65001 dragonflyoss/dfclient:0.3.1 --registry https://index.docker.io



## Verificación


Paso 4: validar Dragonfly
Después de implementar un supernodo y dos dfclients, podemos comenzar a validar si Dragonfly funciona como se esperaba. Puede ejecutar el siguiente comando en los dos nodos dfclient al mismo tiempo para extraer la misma imagen.

docker pull nginx:latest
Puede elegir un nodo dfclient para ejecutar el siguiente comando y verificar si la imagen nginx se distribuye a través de Dragonfly.

docker exec dfclient01 grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log
Si la salida del comando anterior tiene contenido como

2019-03-29 15:49:53.913 INFO sign:96027-1553845785.119 : downloading piece:{"taskID":"00a0503ea12457638ebbef5d0bfae51f9e8e0a0a349312c211f26f53beb93cdc","superNode":"127.0.0.1","dstCid":"127.0.0.1-95953-1553845720.488","range":"67108864-71303167","result":503,"status":701,"pieceSize":4194304,"pieceNum":16}
entonces se demuestra que Dragonfly funciona con éxito.

Si necesita verificar si la imagen se distribuye no solo desde el supernodo, sino también desde otro nodo par (dfclient), puede ejecutar el siguiente comando:

docker exec dfclient01 grep 'downloading piece' /root/.small-dragonfly/logs/dfclient.log | grep -v cdnnode
Si no se muestra ningún resultado, significa que la distribución de imágenes no se ha realizado entre dfclient. De lo contrario, funciona.



