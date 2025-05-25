
Requisitos Previos
Antes de comenzar, asegúrate de tener instaladas las siguientes herramientas en tu sistema Ubuntu 22.04:
    1. Git: Control de versiones.
    2. cURL: Herramienta para transferencias de datos.
    3. Docker: Plataforma de contenedores.
    4. Docker Compose: Herramienta para definir y correr aplicaciones Docker multi-contenedor.
    5. Go (Golang): Lenguaje de programación requerido por Hyperledger Fabric.
Clonar el Repositorio de Merca-link
Clona el repositorio proporcionado por Merca-link y navega al directorio correspondiente:
git clone https://github.com/KeepCodingBlockchain-I/hf-certification-practica.git
cd hf-certification-practica/merca-chain

1. Ejecuta el script oficial para descargar los binarios
curl -sSLO https://raw.githubusercontent.com/hyperledger/fabric/main/scripts/install-fabric.sh && chmod +x install-fabric.sh                                  
./install-fabric.sh b

2. Copiar la carpeta bin y config de fabric-samples al directorio raíz del proyecto de Merca-link
cp -rp ~/fabric-samples/bin ./bin
cp -rp ~/fabric-samples/config ./config

3. Permisos de ejecución:
chmod +x network.sh
cd scripts
sudo chmod +x *
4. Agrega los binarios al PATH
cd ~/hf-certification-practica
echo 'export PATH=$PATH:$HOME/fabric-samples/bin' >> ~/.bashrc
source ~/.bashrc

4. Levantar la red:
cd merca-chain
./network.sh up -ca

5. Verificación:
docker ps 

PRIMER ERROR:
d5dc7fe5d538   hyperledger/fabric-peer:latest      "peer node start"   About a minute ago   Exited (1) 53 seconds ago                                                              peer0.org1.example.com
3da8b736981b   hyperledger/fabric-orderer:latest   "orderer"           About a minute ago   Exited (2) 49 seconds ago                                                              orderer.example.com

6. Identificar los logs:
docker logs d5dc7fe5d538 
docker logs 3da8b736981b

Para peer0.org1.example.com tenemos:
2025-05-12 16:31:13.754 UTC 0005 FATA [nodeCmd] serve -> Error loading secure config for peer (error loading TLS key (open /etc/hyperledger/fabric/tls/server.key: no such file or directory))

Se resuelve cambiando en el fichero compose-test-net.yaml en la sección de volúmenes etc/hypreledger/fabric  por  /etc/hyperledger/fabric.

Para orderer.example.com el error es:
2025-05-12 16:31:17.259 UTC 000e PANI [orderer.common.server] Main -> failed to start admin server: open /var/hyperledger/orderer/tls/servre.crt: no such file or directory
panic: failed to start admin server: open /var/hyperledger/orderer/tls/servre.crt: no such file or directory


Se resuelve cambiando en el fichero compose-test-net.yaml las rutas reales de los certificados /tls/servre.crt  por /tls/server.crt

Tumbar la red y volver a levantarla:
./network.sh down
./network.sh up -ca
docker ps -a

Resultado:
images/1.png
Ejercicio 2
Enunciado: Despliega el chaincode “merca-chaincode”. Aunque el proceso de ciclo de vida del chaincode es satisfactorio, parece que este no se despliega correctamente. Identifica el/los problemas y resuélvelos. Documenta este proceso de investigación y resolución.
Solución:
Crear el canal y desplegar el chaincode:
./network.sh createChannel
./network.sh deployCC -ccn merca-chaincode -ccl javascript -ccv 1.0.0 -ccp ../merca-chaincode

Error al desplegar el chaincode:
Chaincode installation on peer0.org1 has failed
Deploying chaincode failed

 Al ejecutar docker logs 283bba28141f, arroja:
025-05-13 02:00:23.264 UTC 008e ERRO [chaincode.platform] func1 -> docker build failed: Failed to pull hyperledger/fabric-nodeenv:3.1: API error (404): manifest for hyperledger/fabric-nodeenv:3.1 not found: manifest unknown: manifest unknown
2025-05-13 02:00:23.264 UTC 008f ERRO [dockercontroller] buildImage -> Error building image: docker build failed: Failed to pull hyperledger/fabric-nodeenv:3.1: API error (404): manifest for hyperledger/fabric-nodeenv:3.1 not found: manifest unknown: manifest unknown
2025-05-13 02:00:23.265 UTC 0090 ERRO [dockercontroller] buildImage -> Build Output:
Agregar una llave en la línea 89 del fichero merca-chaincode/lib/assetTransfer.js
Crear una actualización del contrato inteligente
./network.sh deployCC -ccn merca-chaincode -ccl javascript -ccv 1.0.1 -ccs 2 -ccp ../merca-chaincode

Solución:
Se agregó la } faltante en la línea. 
Desplegamos de nuevo el contrato y verificamos los logs con: docker ps | grep merca-chaincode
images/2.png

Ejercicio 3
Enunciado: Desde Merca-link nos comentan que los logs de los peers no proporcionan la información deseada. Modifica el logging de la red para que se ajuste a los siguientes requerimientos:
    • Los logs de los orderers deben estar en formato JSON. 
    • Los logs de los peers deben de mostrar la fecha y la hora al final. 
    • Dado que estamos en fase de pruebas, los logs de los peers deben de tener un nivel de logging de DEBUG por defecto. Además, los logs asociados al logger “gossip” tendrán un nivel por defecto de WARNING y los del logger de “chaincode” de INFO para no sobrecargar demasiado los logs. 
Solución:
En el archivo compose/compose-test-net.yaml
1. Formato de log en orderer.example.com
- FABRIC_LOGGING_FORMAT=json
2. FABRIC_LOGGING_SPEC en los peers
- FABRIC_LOGGING_SPEC=debug:gossip=warning:chaincode=info
- FABRIC_LOGGING_FORMAT='%{color} [%{module}] %{shortfunc} -> %{level:.4s} %{id:03x}%{color:reset} %{message} %{time:2006-01-02 15:04:05.000 MST}'
Tumbamos y levantamos de nuevo la red.
Verificamos los cambios con docker logs <CONTANIER_ID>:

images/3.png	
images/4.png

images/5.png

images/6.png

images/7.png	
Ejercicio 4
Enunciado: Los técnicos de Merca-link nos muestran su preocupación acerca de las claves criptográficas, que se almacenan en texto plano dentro de los directorios de la red. Como una primera medida de securización de las claves y de las operaciones criptográficas sugieren el acoplamiento de un HSM a las CAs, utilizándose softhsm2 para esta fase de pruebas y configurando un token diferente para cada CA.
Solución:
Instalación y configuración de SoftHSM2
sudo apt install softhsm2
export SOFTHSM2_CONF=/etc/softhsm2.conf

Inicializar tokens
cd /home/cindy
mkdir tokens

# SoftHSM v2 configuration file
directories.tokendir = /home/cindy/tokens
objectstore.backend = file
# ERROR, WARNING, INFO, DEBUG
log.level = ERROR
# If CKF_REMOVABLE_DEVICE flag should be set
slots.removable = false

softhsm2-util --init-token --slot 0 --label fabric
softhsm2-util --init-token --slot 1 --label fabric1
softhsm2-util --init-token --slot 2 --label fabric2

images/8.png	
Reconstrucción de imagen de fabric-ca-server
docker image ls
docker image rm <ID_IMAGE>	

Remover las imágenes del docker
images/9.png	
Clonar el repsositorio
git clone https://github.com/hyperledger/fabric-ca.git 
cd fabric-ca
Reconstruir la imagen y verificamos la imagen recién creada
images/10.png	
Reconfiguración de Fabric CA
    • /organizations/fabric-ca/org1/fabric-ca-server-config.yaml
    • /organizations/fabric-ca/org2/fabric-ca-server-config.yaml 
    • /organizations/fabric-ca/ordererOrg/fabric-ca-server-config.yaml 
Reiniciamos la red pero no levanta con ca:
images/11.png	
Como se puede observar en los archivos de configuración de la ca para cada organización se realizaron los cambios y configuraciones necesarios, para la ca no se levanta.
images/12.png

Ejercicio 5
Enunciado: Así mismo, explícales cómo configurar y utilizar el fabric-ca-client con el softHSM configurado. Registra, a modo de prueba, un usuario de nombre “merca-admin” y contraseña “merka-12345”, que tenga capacidad para registrar usuarios clientes y nuevos peers.
Solución:
Como no logré los contenedores CA no es posble hacer el register:
images/13.png
Ejercicio 6
Enunciado: Los merca-ingenieros de Merca-link consideran que tener herramientas de monitorización a su disposición agilizaría estas tareas de administración en futuras ocasiones. Despliega sendas instancias de Prometheus y Grafana para satisfacer estas necesidades.
Solución:
Prometheus
images/14.png	
images/15.png
