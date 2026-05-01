# Segundo Laboratorio 

## Procedimiento del desarrollo del laboratorio

## 1. Planeación e implementación de la arquitectura
<img width="797" height="1028" alt="image" src="https://github.com/user-attachments/assets/62a2b58f-8362-4e93-8d99-3bbf3302237a" />

Para el desarrollo del laboratorio se inició con el diseño de una arquitectura segmentada en dos subredes, administradas desde una máquina virtual principal con Debian, la cual cumple funciones de monitoreo, visualización, análisis de tráfico y administración general.

### 1.1 Configuración del nodo administrador
### Tabla de instancias creadas:
<img width="1352" height="366" alt="image" src="https://github.com/user-attachments/assets/d00c9597-9678-49e8-bc50-65f4ef78adef" />

Se creó una máquina virtual con Debian y se configuraron tres interfaces de red:

* **eth0:** interfaz de gestión con acceso a internet mediante NAT o modo puente.
* **eth1:** interfaz configurada como trunk 802.1Q para manejar múltiples VLAN.
* **eth2:** interfaz destinada a captura pasiva mediante puerto espejo SPAN.

Posteriormente se instalaron las herramientas requeridas:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install grafana zabbix-server-mysql zabbix-frontend-php \
mysql-server wireshark iperf3 softflowd pmacct -y
```

---

## 2. Configuración del Switch Cisco 2960

Se realizó la segmentación lógica mediante VLANs:

* VLAN 10 → 192.168.10.0/24
* VLAN 20 → 192.168.20.0/24

### Creación de VLANs

```cisco
enable
configure terminal

vlan 10
name SUBRED1
exit

vlan 20
name SUBRED2
exit
```


### Asignación de puertos

```cisco
interface range fa0/1-12
switchport mode access
switchport access vlan 10

interface range fa0/13-22
switchport mode access
switchport access vlan 20
```
<img width="538" height="489" alt="image" src="https://github.com/user-attachments/assets/3e51eb0c-0bbe-41e9-a06a-94642ca05935" />

### Configuración del puerto trunk

```cisco
interface g0/24
switchport mode trunk
switchport trunk allowed vlan 10,20
```
<img width="538" height="489" alt="image" src="https://github.com/user-attachments/assets/3e51eb0c-0bbe-41e9-a06a-94642ca05935" />

### Configuración SPAN

```cisco
monitor session 1 source vlan 10
monitor session 1 source vlan 20
monitor session 1 destination interface g0/23
```
<img width="502" height="600" alt="image" src="https://github.com/user-attachments/assets/6afbbcdc-2b79-4fe9-a216-7639802509e5" />

Con esto todo el tráfico de ambas VLAN fue replicado hacia la interfaz de captura.

---

## 3. Configuración Router-on-a-Stick en Debian

Se habilitó el enrutamiento entre VLANs desde Debian usando subinterfaces.

```bash
sudo modprobe 8021q

sudo ip link add link eth1 name eth1.10 type vlan id 10
sudo ip link add link eth1 name eth1.20 type vlan id 20

sudo ip addr add 192.168.10.1/24 dev eth1.10
sudo ip addr add 192.168.20.1/24 dev eth1.20

sudo ip link set eth1.10 up
sudo ip link set eth1.20 up
```

### Activación del forwarding

```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```
<img width="1118" height="810" alt="image" src="https://github.com/user-attachments/assets/66de19e8-f7f1-40c0-a816-e95641569335" />

Esto permitió comunicación entre subredes.

### Reglas del Security group

<img width="1231" height="516" alt="image" src="https://github.com/user-attachments/assets/af786a7d-ff06-4265-a2da-1a0f36e7bebf" />

---

## 4. Implementación de máquinas virtuales y contenedores

### Subred 1

Se desplegaron:

* Máquina virtual Arch Linux.
* Máquina virtual Rocky Linux.
* Contenedor Fedora.

### Subred 2

Se desplegaron:

* Contenedor Alpine.
* Contenedor Kali Linux.
* Máquina virtual Ubuntu.

Cada host fue configurado con:

```bash
ip addr add 192.168.X.X/24 dev eth0
ip route add default via 192.168.X.1
```

Se verificó conectividad con:

```bash
ping
traceroute
```

---

# 5. Implementación de NetFlow, sFlow e IPFIX

## Softflowd para generación de NetFlow

```bash
sudo softflowd -i eth2 -n 127.0.0.1:2055 -v9
```

Se exportaron flujos NetFlow v9 hacia pmacct.

## Configuración de pmacct

Archivo:

```bash
sudo nano /etc/pmacct/collector.conf
```

Configuración:

```conf
daemonize: true
plugins: mysql
sql_db: pmacct_db
sql_table: flows
nfacctd_port: 2055
```

Ejecución:

```bash
sudo pmacctd -f /etc/pmacct/collector.conf
```

### Relevancia de estas tecnologías

Se implementaron porque permiten:

* Medir flujos de tráfico.
* Identificar conversaciones entre hosts.
* Analizar consumo de ancho de banda.
* Detectar congestión.
* Exportar métricas para Grafana.
* Comparar diferencias entre NetFlow, sFlow e IPFIX.

---

# 6. Instalación y configuración de Zabbix

Creación de base de datos:

```sql
create database zabbix;
```

Importación del esquema:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u root -p zabbix
```
<img width="1007" height="564" alt="image" src="https://github.com/user-attachments/assets/a7c8718d-0cc4-48d4-bf46-8997c7c141b3" />

Inicio servicios:

```bash
sudo systemctl enable zabbix-server
sudo systemctl start zabbix-server
```
<img width="865" height="541" alt="image" src="https://github.com/user-attachments/assets/2516e2d9-8305-4ef4-8a01-9cdbe18746db" />

Se añadieron hosts de ambas VLAN para monitorear:

* CPU
* Memoria
* Interfaces
* Tráfico
* Disponibilidad

---

# 7. Integración de Grafana

Acceso:

```bash
http://localhost:3000
```
<img width="1374" height="646" alt="image" src="https://github.com/user-attachments/assets/40b2ec32-3f03-49cc-bce3-41abe43a1676" />

Se agregaron dos fuentes de datos:

## Fuente MySQL

Para consultar flujos de pmacct.

## Plugin Zabbix

```bash
grafana-cli plugins install alexanderzobnin-zabbix-app
```

Se construyó un dashboard con:

* Tráfico por segundo.
* Top conversaciones.
* Uso de CPU.
* Consumo de memoria.
* Interfaces activas.
* Flujo entre VLAN.

---

# 8. Generación de tráfico con iPerf3

<img width="1018" height="759" alt="image" src="https://github.com/user-attachments/assets/4fb2baba-7d24-4f1a-b738-523882d3efb6" />

## Servidor

```bash
iperf3 -s
```

## Cliente TCP

```bash
iperf3 -c 192.168.10.20 -t 60
```

## Cliente UDP

```bash
iperf3 -c 192.168.10.20 -u -b 50M -t 60
```

Se variaron:

* Protocolo.
* Duración.
* Puertos.
* Ancho de banda.

Con esto se observó cómo los flujos aparecían en Grafana y en las capturas.

---

# 9. Captura y análisis con Wireshark

<img width="1336" height="928" alt="image" src="https://github.com/user-attachments/assets/144eae4d-59f8-43d5-bd6d-541b656853bf" />

Captura:

```bash
sudo wireshark
```

Interfaz utilizada:

```bash
eth2
```

Filtros usados:

```wireshark
ip.addr==192.168.10.10
```

```wireshark
tcp.port==5201
```

```wireshark
udp.port==5201
```

## Resultados observados

Se analizaron:

* Paquetes TCP y UDP.
* Throughput.
* Jitter.
* Retransmisiones.
* Pérdida de paquetes.
* Ventanas TCP.

### Qué se obtiene con Wireshark

Wireshark permitió:

* Capturar paquetes en tiempo real.
* Ver encabezados.
* Analizar protocolos.
* Ver efectos de iPerf3 sobre el tráfico.
* Correlacionar congestión con desempeño.

Cuando se ejecuta iPerf se generan grandes volúmenes de tráfico que permiten medir rendimiento, y estos se reflejan tanto en flujos exportados como en capturas.
### Conexion SSH de cada instancia
<img width="1363" height="634" alt="image" src="https://github.com/user-attachments/assets/26878439-a043-4584-b0c6-cc13b6b5b167" />

---

# 10. Diferencias observadas entre máquinas virtuales y contenedores

## Máquinas virtuales

Se observó:

* Mayor consumo de recursos.
* Sistema operativo completo.
* Más aislamiento.
* Arranque más lento.

## Contenedores

Se observó:

* Menor consumo.
* Inicio rápido.
* Menor overhead.
* Compartición del kernel.

Estas diferencias fueron visibles durante instalación y pruebas de tráfico.

---

# Segundo Punto – Relación tráfico físico y red

## 11. Topología física

Se conectaron tres equipos:

* Servidor de video.
* Cliente monitor.
* Cliente generador de tráfico.

Direcciones:

```text
Servidor: 192.168.1.10
Monitor: 192.168.1.20
Cliente tráfico: 192.168.1.30
Gateway: 192.168.1.1
```

---

## 12. Implementación de YOLO

Se creó entorno virtual:

<img width="1298" height="550" alt="image" src="https://github.com/user-attachments/assets/1e8bfa87-03ec-4a7a-ad9d-65a981a556cd" />


```bash
python3 -m venv venv
source venv/bin/activate
```

Dependencias:

```bash
pip install ultralytics opencv-python
```

Ejecución del modelo:

```bash
python conmutacion.py
```

El script permitió detectar objetos y correlacionar FPS con tráfico capturado.

---

## 13. Pruebas de rendimiento con iPerf3

<img width="1278" height="521" alt="image" src="https://github.com/user-attachments/assets/706ca812-f372-4883-b867-ac3289d61fa3" />


Servidor:

```bash
iperf3 -s
```

Cliente TCP:

```bash
iperf3 -c 192.168.1.10
```

Cliente UDP:

```bash
iperf3 -c 192.168.1.10 -u -b 50M
```

Se evaluó:

* Saturación.
* Pérdida.
* Jitter.
* Impacto sobre FPS de YOLO.

---

## 14. Captura RTSP y RTP con Wireshark

Filtros:

```wireshark
rtsp
```

```wireshark
rtp
```

Se analizaron mensajes:

* SETUP
* PLAY
* TEARDOWN

En RTP se observaron:

* Sequence Number
* Timestamp
* Flujo de fotogramas

Se correlacionaron con detecciones de YOLO.

---

## 15. Calidad de servicio y congestión

Con pruebas UDP a 50 Mbps se evidenció:

* Aumento de pérdida de paquetes.
* Variación del jitter.
* Reducción del rendimiento del procesamiento.

Al cambiar a un modelo YOLO más pesado:

```bash
yolov8l.pt
```

Se observó reducción de FPS y aumento del uso de recursos.

---

## 16. Integración con equipos reales

Ejemplo de limitación de tráfico en Cisco:

```cisco
policy-map VIDEO-LIMIT
 class class-default
  police 10000000

interface g0/1
 service-policy output VIDEO-LIMIT
```

Se verificó con Wireshark observando:

* Throughput limitado.
* Reducción del bitrate.
* Comportamiento del flujo RTP.

---

# Conclusiones del procedimiento

Durante el procedimiento se logró:

1. Construir una arquitectura segmentada con VLAN.
2. Integrar Grafana y Zabbix para monitoreo.
3. Implementar NetFlow, sFlow e IPFIX.
4. Analizar tráfico con Wireshark.
5. Generar pruebas de carga con iPerf3.
6. Relacionar visión computacional y teletráfico mediante YOLO.
7. Validar comportamiento de red en entornos virtualizados y físicos.
----
## Preguntas del laboratorio
### Describir paso a paso el desarrollo de la arquitectura y su conformación.

La arquitectura se implementó en dos capas. En AWS se desplegaron 4 instancias EC2 dentro de una VPC con dos subnets que simulan las VLAN 10 (10.0.10.0/24) y VLAN 20 (10.0.20.0/24). La VM Admin (Debian, t3.small) actúa como nodo central con Grafana, Zabbix y pmacct. En la máquina local se ejecutaron 3 contenedores Docker (Fedora, Alpine, Kali Linux) y un switch Cisco 2960 simulado en GNS3 con VLANs, puerto trunk y puerto SPAN. Ambas capas se interconectan mediante un túnel WireGuard (UDP 51820), permitiendo que los contenedores locales se comuniquen con las VMs de AWS como si estuvieran en la misma red. El flujo de monitoreo es: tráfico → SPAN → softflowd/pmacct → NetFlow → MariaDB → Grafana.

### ¿Cuál es la relevancia en el proyecto de NetFlow, sFlow e IPFIX?

NetFlow (creado por Cisco) exporta registros de flujos IP con información de origen, destino, protocolo, puertos y bytes transferidos. sFlow usa muestreo estadístico de paquetes en tiempo real, ideal para redes de alta velocidad. IPFIX es el estándar abierto basado en NetFlow v9 que permite exportar metadatos flexibles. En este laboratorio, pmacct recolecta los tres protocolos simultáneamente (puertos 2055, 6343 y 4739), almacena los flujos en MariaDB y Grafana los visualiza. Esto permite al administrador ver en tiempo real quién habla con quién, cuánto tráfico genera cada host y detectar anomalías, sin necesidad de capturar paquetes completos como hace Wireshark.

###  ¿Qué se obtiene con Wireshark? ¿Qué pasa cuando se ejecuta iPerf?

Wireshark captura y decodifica paquetes en tiempo real mostrando la jerarquía de protocolos (Ethernet → IP → TCP/UDP → RTSP/RTP). Con el flujo de video activo se observan: mensajes RTSP en puerto 554 TCP (OPTIONS, DESCRIBE, SETUP, PLAY, TEARDOWN) para la señalización, y paquetes RTP en puertos UDP altos para el transporte real del video con número de secuencia y timestamp. Cuando se activa iPerf3, el ancho de banda disponible disminuye: en modo TCP se observa el mecanismo de ventana deslizante (control de flujo) expandiéndose y contrayéndose; en modo UDP no hay control de flujo y aparece pérdida de paquetes RTP medible en Wireshark como gaps en los números de secuencia, lo que degrada directamente los FPS de YOLO.

### ¿Cuáles son las diferencias entre máquinas virtuales y contenedores al instalar los sistemas operativos?

Las máquinas virtuales (Arch, Rocky, Ubuntu, Debian en AWS) requieren una imagen de disco completa con kernel propio, bootloader y todos los servicios del sistema. Su arranque toma 20-60 segundos y consumen varios GB de RAM independientemente de lo que ejecuten. Los contenedores Docker (Fedora, Alpine, Kali) comparten el kernel del host, solo incluyen las librerías de usuario necesarias y arrancan en menos de 1 segundo. Alpine ocupa apenas 5MB de imagen base versus los varios GB de una VM. La diferencia práctica en el laboratorio fue notoria: las VMs en AWS tardaron varios minutos en estar operativas mientras los contenedores Docker locales estuvieron listos instantáneamente. El aislamiento de las VMs es completo (hardware virtualizado), mientras los contenedores comparten el kernel pero tienen namespaces de red, proceso y filesystem separados.

El laboratorio permitió correlacionar conceptos de conmutación, monitoreo, teletráfico, análisis de paquetes y calidad de servicio en una única solución integrada.
