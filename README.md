# Segundo Laboratorio 

## Procedimiento del desarrollo del laboratorio

## 1. Planeación e implementación de la arquitectura

Para el desarrollo del laboratorio se inició con el diseño de una arquitectura segmentada en dos subredes, administradas desde una máquina virtual principal con Debian, la cual cumple funciones de monitoreo, visualización, análisis de tráfico y administración general.

### 1.1 Configuración del nodo administrador

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

Esto permitió comunicación entre subredes.

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

Inicio servicios:

```bash
sudo systemctl enable zabbix-server
sudo systemctl start zabbix-server
```

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

El laboratorio permitió correlacionar conceptos de conmutación, monitoreo, teletráfico, análisis de paquetes y calidad de servicio en una única solución integrada.
