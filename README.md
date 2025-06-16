# Energy Monitoring System

Este proyecto implementa un sistema de monitoreo energético basado en Tasmota + MQTT + InfluxDB + Grafana. Puedes ejecutarlo tanto con Docker Compose como con Kubernetes (K3s). A continuación se explican ambos métodos.

---

## 📦 Infraestructura del sistema

El sistema está compuesto por:

* **Mosquitto (MQTT Broker)**: Recibe datos desde el dispositivo Tasmota.
* **InfluxDB (1.8)**: Base de datos de series temporales.
* **Telegraf**: Suscribe a MQTT y reenvía datos a InfluxDB.
* **Grafana**: Visualiza las métricas energéticas.
* **Dispositivo Tasmota**: Sonoff POWR316D con firmware Tasmota 14.6.0, configurado para publicar en el tópico MQTT `tele/tasmota_E8D2BC/SENSOR`.

---

## 🐳 Ejecución con Docker Compose

### 1. Requisitos

* Docker
* docker-compose

### 2. Estructura esperada

```
.
├── docker-compose.yml
├── mosquitto/
│   └── config/
│       └── mosquitto.conf
├── influxdb/     # Volumen persistente
├── grafana/      # Volumen persistente
└── telegraf/
    └── telegraf.conf
```

### 3. Levantar los servicios

```bash
docker-compose up -d
```

* Mosquitto escucha en `1883`
* Grafana: [http://localhost:3000](http://localhost:3000) (admin/admin)
* InfluxDB: [http://localhost:8086](http://localhost:8086)

---

## ☸️ Ejecución en Kubernetes (K3s)

### 1. Requisitos

* K3s instalado en un equipo como un Intel NUC (NUC7CJYH con 8GB RAM)
* `kubectl` configurado

### 2. Despliegue

```bash
kubectl apply -f energy-monitoring.yaml
```

Esto desplegará:

* Namespace `energy-monitoring`
* Deployments para Mosquitto, InfluxDB, Telegraf y Grafana
* ConfigMaps con `telegraf.conf` y `mosquitto.conf`
* PVCs para InfluxDB y Grafana
* Servicios:

  * Mosquitto: `LoadBalancer` en puerto 1883 (accesible desde la red local)
  * InfluxDB y Grafana: `NodePort`

### 3. Acceso a servicios

* **Grafana**: http\://\<IP\_NUC>:32300
* **InfluxDB**: http\://\<IP\_NUC>:32086
* **Mosquitto**: mqtt://\<IP\_NUC>:1883

---

## ⚙️ Configuración del dispositivo Tasmota

Desde la interfaz web de Tasmota, en **Configuration → Configure MQTT**:

* Host: `<IP_NUC>`
* Port: `1883`
* Topic: `tasmota_E8D2BC`
* Full Topic: `%prefix%/%topic%/`
* User: (vacío)
* Password: (vacío)

También puedes configurar desde la consola Tasmota:

```bash
Backlog MqttUser; MqttPassword; MqttHost 192.168.110.144; MqttPort 1883; MqttClient DVES_E8D2BC; Topic tasmota_E8D2BC; FullTopic %prefix%/%topic%/
```

---

## 📊 Visualización en Grafana

1. Accede a Grafana en http\://\<IP\_NUC>:32300
2. Fuente de datos: InfluxDB

   * URL: `http://influxdb:8086`
   * Base de datos: `tasmota`
   * Usuario/contraseña: `admin / admin123`
3. Crea paneles usando la métrica `tasmota_energia`, por ejemplo:

```sql
SELECT "ENERGY_Power" FROM "tasmota_energia"
```

---

## 🧠 Créditos

Este sistema fue desarrollado e implementado por [@flexomeno](https://github.com/flexomeno) como parte de un entorno doméstico de monitoreo energético usando software libre y contenedores.

---

## 📄 Licencia

MIT License
