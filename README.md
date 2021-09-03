

# Mosquitto con TLS
- [Mosquitto con TLS](#mosquitto-con-tls)
  - [Crear los certificados](#crear-los-certificados)
    - [Correr el siguiente script](#correr-el-siguiente-script)
  - [Configurar mosquitto instalado en sistema](#configurar-mosquitto-instalado-en-sistema)
    - [Editar el archivo /etc/mosquitto/mosquitto.conf](#editar-el-archivo-etcmosquittomosquittoconf)
    - [Guardar los certificados en /etc/mosquitto/certs](#guardar-los-certificados-en-etcmosquittocerts)
    - [Reiniciar el servicio](#reiniciar-el-servicio)
    - [Verificación](#verificación)
  - [Configuración de mosquitto broker con docker](#configuración-de-mosquitto-broker-con-docker)
    - [Contenido del archivo docker-compose.yml](#contenido-del-archivo-docker-composeyml)
    - [Iniciar el contenedor](#iniciar-el-contenedor)
    - [Verificación mosquitto con docker](#verificación-mosquitto-con-docker)
  - [probar comunicacion](#probar-comunicacion)
    - [Certificados para Thingsboard](#certificados-para-thingsboard)
## Crear los certificados
fuente:
https://medium.com/himinds/mqtt-broker-with-secure-tls-communication-on-ubuntu-18-04-lts-and-an-esp32-mqtt-client-5c25fd7afe67



### Correr el siguiente script
```sh
#!/bin/bash

IP="192.168.1.42"
SUBJECT_CA="/C=AR/ST=Argentina/L=Rosario/O=FiUBA/OU=CA/CN=$IP"
SUBJECT_SERVER="/C=AR/ST=Argentina/L=Rosario/O=FiUBA/OU=Server/CN=$IP"
SUBJECT_CLIENT="/C=AR/ST=Argentina/L=Rosario/O=FiUBA/OU=Client/CN=$IP"


function generate_CA () {
   echo "$SUBJECT_CA"
   openssl req -x509 -nodes -sha256 -newkey rsa:2048 -subj "$SUBJECT_CA"  -days 365 -keyout ca.key -out ca.crt
}

function generate_server () {
   echo "$SUBJECT_SERVER"
   openssl req -nodes -sha256 -new -subj "$SUBJECT_SERVER" -keyout server.key -out server.csr
   openssl x509 -req -sha256 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365
}

function generate_client () {
   echo "$SUBJECT_CLIENT"
   openssl req -new -nodes -sha256 -subj "$SUBJECT_CLIENT" -out client.csr -keyout client.key 
   openssl x509 -req -sha256 -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365
}

function copy_keys_to_broker () {
   cp ca.crt ../mosquitto/certs/
   cp server.crt ../mosquitto/certs/
   cp server.key ../mosquitto/certs/
}

generate_CA
generate_server
generate_client
#copy_keys_to_broker
```

## Configurar mosquitto instalado en sistema
### Editar el archivo /etc/mosquitto/mosquitto.conf
```sh
listener 8883
cafile /mosquitto/certs/ca.crt
keyfile /mosquitto/certs/server.key
certfile /mosquitto/certs/server.crt
require_certificate true
```
### Guardar los certificados en /etc/mosquitto/certs
- ca.crt
- server.crt
- server.key

### Reiniciar el servicio
```sh
systemctl restart mosquitto
```
### Verificación

Con systemctl vemos si el servicio está activo

```sh 
systemctl status mosquitto
```

Con journal, verificamos que no hay errores ni advertencias
```sh
journalctl -u mosquitto
```

Con netstat verificamos que existe un servicio que escucha el puerto 8883
```sh
netstat -nap | grep 8883
```

## Configuración de mosquitto broker con docker

### Contenido del archivo docker-compose.yml
```sh
version: '3'

services:

  mosquitto:
    # service header
    image:                  eclipse-mosquitto:latest
    hostname:               mosquitto
    container_name:         mosquitto
    
    # project labels 
    labels: 
      project:              "DAIoT" 
      day:                  "Sep 2021" 
      mantainer:            "Marcelo Castello"
      
    # service specific settings
    volumes:
      -                     ./config:/mosquitto/config
      -                     ./certs:/mosquitto/certs
    networks:
      -                     broker-network 
    expose:
      -                     "8883"
    ports:
      -                     "8883:8883"

networks:
  broker-network:
    driver:                 bridge
```
### Iniciar el contenedor
```sh
docker-compose up
```
### Verificación mosquitto con docker

Con netstat verificamos que existe un servicio que escucha el puerto 8883
```sh
netstat -nap | grep 8883
```

## probar comunicacion

Suscripción para escuchar todos los mensajes
```sh
mosquitto_sub -h '192.168.1.42' -t 'topico' --cafile ca.crt  -p 8883 --cert client.crt --key client.key -v
```

Envío de mensajes
```sh
mosquitto_pub -h '192.168.1.42' -t 'device/command'  -m '{"Device":"DAIoT01","Command":"version"}' -p 8883 --cafile ca.crt --cert client.crt --key client.key
```


### Certificados para Thingsboard

Instalar herramientas

apt-get install default-jre

Configuración del script de generación, archivo keygen.properties

```sh
DOMAIN_SUFFIX="192.168.1.25"
ORGANIZATIONAL_UNIT=MUNICIPALIDAD
ORGANIZATION=MUNICIPALIDAD
CITY=Rosario
STATE_OR_PROVINCE=SantaFe
TWO_LETTER_COUNTRY_CODE=AR 
SERVER_KEYSTORE_PASSWORD=passwdserver
SERVER_KEY_PASSWORD=passwdserver
SERVER_KEY_ALIAS="serveralias"
SERVER_FILE_PREFIX="mqttserver"
SERVER_KEYSTORE_DIR="/etc/thingsboard/conf"
CLIENT_KEYSTORE_PASSWORD=keystorepassclient
CLIENT_KEY_PASSWORD=keystorepassclient
CLIENT_KEY_ALIAS="clientalias"
CLIENT_FILE_PREFIX="mqttclient"
```
Correr el script de generación de los certificados

```sh
./server.keygen.sh
```
Esto genera los siguientes archivos:

mqttserver.cer
mqttserver.jks
mqttserver.pub.pem
mqttserver.jks -> certificado usado en el servidor (Thingboard)

mqttserver.pub.pem -> certificado usado en los clientes



