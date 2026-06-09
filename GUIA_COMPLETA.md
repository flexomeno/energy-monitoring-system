# Guía completa: Energy Monitoring System

Sistema de monitoreo energético con **Tasmota → MQTT (Mosquitto) → Telegraf → InfluxDB → Grafana**, desplegado en **Kubernetes (k3d)** desde un Mac.

Para una referencia rápida del proyecto, consulta el [README.md](README.md) principal.

---

## Tabla de contenidos

1. [Requisitos en el Mac](#1-requisitos-en-el-mac)
2. [Referencia del dispositivo](#2-referencia-del-dispositivo)
3. [Flashear Tasmota en el Sonoff POWR316D](#3-flashear-tasmota-en-el-sonoff-powr316d)
4. [Despliegue del stack en Kubernetes (k3d)](#4-despliegue-del-stack-en-kubernetes-k3d)
5. [Configuración del dispositivo Tasmota](#5-configuración-del-dispositivo-tasmota)
6. [Validaciones paso a paso](#6-validaciones-paso-a-paso)
7. [Configuración de Grafana](#7-configuración-de-grafana)
8. [Datos recopilados y unidades](#8-datos-recopilados-y-unidades)
9. [Arquitectura del flujo de datos](#9-arquitectura-del-flujo-de-datos)
10. [Credenciales del sistema](#10-credenciales-del-sistema)
11. [Comandos de mantenimiento](#11-comandos-de-mantenimiento)
12. [Troubleshooting rápido](#12-troubleshooting-rápido)

---

## 1. Requisitos en el Mac

### Software

| Herramienta | Para qué | Instalación |
|-------------|----------|-------------|
| **Docker Desktop** | Motor de contenedores para k3d | [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/) |
| **k3d** | Cluster Kubernetes local ligero | `brew install k3d` |
| **kubectl** | CLI de Kubernetes | `brew install kubectl` |
| **Git** | Clonar el repo | `brew install git` |
| **mosquitto** (opcional) | Pruebas MQTT manuales | `brew install mosquitto` |

### Hardware de red

- Mac conectado a la **misma red WiFi/LAN** que el Sonoff.
- IP local fija o estable en el Mac (ejemplo: `192.168.110.165`).
- Puertos libres: **1883** (MQTT), **3000** (Grafana vía port-forward).

### Recursos recomendados

- **RAM:** 8 GB mínimo (Docker + k3d + 4 contenedores).
- **Disco:** ~2 GB para imágenes Docker (influxdb, grafana, telegraf, mosquitto).

### Verificar instalación

```bash
docker --version
k3d version
kubectl version --client
```

---

## 2. Referencia del dispositivo

| Campo | Valor |
|-------|-------|
| **Modelo** | Sonoff POW Elite 16A (**POWR316D**) |
| **Fabricante** | SONOFF / ITEAD |
| **MCU** | ESP32 |
| **Chip de medición** | CSE7766 |
| **Firmware** | Tasmota 14.6.0 (`release-tasmota32`) |
| **Capacidad** | Hasta 16 A |
| **Pantalla** | TM1621 (vuelve a funcionar tras aplicar template) |
| **ID del dispositivo** | `E8D2BC` (sufijo MAC; puede variar en tu unidad) |
| **Topic MQTT** | `tasmota_E8D2BC` |
| **Tópico de telemetría** | `tele/tasmota_E8D2BC/SENSOR` |

### Documentación para flashear Tasmota

| Recurso | Enlace |
|---------|--------|
| **Tasmota — Getting Started** (guía oficial) | [tasmota.github.io/docs/Getting-Started](https://tasmota.github.io/docs/Getting-Started/) |
| **Tasmota Web Installer** (método recomendado para ESP32) | [tasmota.github.io/install](https://tasmota.github.io/install/) |
| **Flasheo ESP32** (requisitos de alimentación, binarios) | [tasmota.github.io/docs/ESP32](https://tasmota.github.io/docs/ESP32/) |
| **Template POWR316D** (GPIO + configuración) | [templates.blakadder.com/sonoff_POWR316D](https://templates.blakadder.com/sonoff_POWR316D.html) |
| **Guía paso a paso con fotos** | [bangertech.de/sonoff-pow-elite](https://bangertech.de/sonoff-pow-elite/) |
| **Experiencia con Web Installer** | [splitbrain.org — Liberating the Sonoff Power Elite](https://www.splitbrain.org/blog/2023-08/06-liberating_the_sonoff_power_elite) |

---

## 3. Flashear Tasmota en el Sonoff POWR316D

### Resumen del proceso

1. Usar un **adaptador USB-Serial 3.3V** de calidad (≥ 500 mA). Evitar clones FTDI baratos.
2. **Nunca** conectar el adaptador serial con el dispositivo conectado a la red eléctrica (220V).
3. Abrir la carcasa y localizar los pads de programación (TX, RX, GND, 3.3V).
4. Conectar **TX ↔ RX** cruzados.
5. Entrar en modo flash: mantener pulsado el botón del dispositivo al conectar/iniciar el flasheo.
6. Flashear con el **[Web Installer](https://tasmota.github.io/install/)** → elegir **Tasmota32** (no ESP8266).
7. Desconectar cables serial y alimentar por red eléctrica.
8. Conectar al AP WiFi de Tasmota y configurar tu red.
9. En **Configuration → Configure Other**, activar template y pegar:

```json
{"NAME":"Sonoff POWR316D","GPIO":[32,0,0,0,0,576,0,0,0,224,9280,0,3104,0,320,0,0,0,0,0,0,9184,9248,9216,0,0,0,0,0,0,0,0,0,0,0,0],"FLAG":0,"BASE":1}
```

O usar **Configuration → Auto configuration** y seleccionar **Sonoff POWR316D** si está en la lista.

10. Calibrar medición de energía si los valores no son precisos (comandos `VoltageSet`, `CurrentSet`, etc. en consola Tasmota).

> Tras flashear, la pantalla puede no mostrar nada hasta aplicar el template correcto. Busca el AP WiFi de Tasmota para confirmar que el firmware arrancó.

---

## 4. Despliegue del stack en Kubernetes (k3d)

### 4.1 Clonar el repositorio

```bash
git clone https://github.com/flexomeno/energy-monitoring-system.git
cd energy-monitoring-system
```

### 4.2 Personalizar el tópico MQTT (si tu dispositivo tiene otro ID)

Edita `kubernetes_deployment`, en el ConfigMap de Telegraf:

```yaml
topics = ["tele/tasmota_E8D2BC/SENSOR"]
```

Cambia `E8D2BC` por el sufijo de tu Sonoff (visible en la web de Tasmota → **Information**).

Si usas Docker Compose, edita el mismo tópico en `telegraf/telegraf.conf`.

### 4.3 Crear el cluster k3d con MQTT expuesto

```bash
k3d cluster create energy-monitoring -p "1883:1883@loadbalancer"
```

Esto mapea el puerto **1883** del broker MQTT al Mac, para que Tasmota pueda conectar desde la red local.

### 4.4 Desplegar el manifiesto

```bash
kubectl apply -f kubernetes_deployment
```

Esto crea:

- Namespace `energy-monitoring`
- Deployments: Mosquitto, InfluxDB, Telegraf, Grafana
- ConfigMaps con `telegraf.conf` y `mosquitto.conf`
- PVCs para InfluxDB y Grafana
- Services (Mosquitto LoadBalancer :1883, InfluxDB y Grafana NodePort)

### 4.5 Esperar a que los pods estén listos

```bash
kubectl get pods -n energy-monitoring -w
```

Espera hasta ver **4/4 pods** en `Running` y `READY 1/1`. La primera vez tarda ~1–2 minutos (descarga de imágenes).

```bash
kubectl wait --for=condition=ready pod --all -n energy-monitoring --timeout=180s
```

### Alternativa: Docker Compose

Si prefieres no usar Kubernetes:

```bash
mkdir -p mosquitto/data mosquitto/log influxdb grafana
docker-compose up -d
```

- Mosquitto: `localhost:1883`
- Grafana: http://localhost:3000
- InfluxDB: http://localhost:8086

---

## 5. Configuración del dispositivo Tasmota

En **Configuration → Configure MQTT**:

| Campo | Valor |
|-------|-------|
| **Host** | IP del Mac (o NUC) en la LAN, ej. `192.168.110.165` |
| **Port** | `1883` |
| **Client** | `DVES_E8D2BC` (o dejar default) |
| **Topic** | `tasmota_E8D2BC` |
| **Full Topic** | `%prefix%/%topic%/` |
| **User** | vacío (Mosquitto permite anónimos) |
| **Password** | vacío |

Por consola Tasmota:

```bash
Backlog MqttUser ; MqttPassword ; MqttHost 192.168.110.165 ; MqttPort 1883 ; Topic tasmota_E8D2BC ; FullTopic %prefix%/%topic%/
```

### Telemetría (opcional, recomendado)

Por defecto Tasmota publica cada **300 s**. Para datos más frecuentes:

```bash
TelePeriod 10
```

Publica en `tele/tasmota_E8D2BC/SENSOR` cada 10 segundos (coincide con el intervalo de Telegraf).

---

## 6. Validaciones paso a paso

### 6.1 Pods en ejecución

```bash
kubectl get pods -n energy-monitoring
```

Esperado: 4 pods `1/1 Running`.

> Si aparece `pod does not have a host assigned`, el pod aún está arrancando. Espera a `Running 1/1` antes de ejecutar `kubectl exec`.

### 6.2 Servicios y almacenamiento

```bash
kubectl get svc,pvc -n energy-monitoring
```

- PVCs `influxdb-pvc` y `grafana-pvc` en estado **Bound**
- Mosquitto tipo **LoadBalancer** en puerto 1883

### 6.3 Tasmota conectado a Mosquitto

```bash
kubectl logs -n energy-monitoring -l app=mosquitto --tail=20
```

Busca líneas como:

```
New client connected ... as DVES_E8D2BC
New client connected ... as Telegraf-Consumer-...
```

### 6.4 Telegraf conectado

```bash
kubectl logs -n energy-monitoring -l app=telegraf --tail=20
```

Busca:

```
[inputs.mqtt_consumer] Connected [tcp://mosquitto:1883]
```

El warning inicial de InfluxDB (`connection refused`) al arrancar es normal si Telegraf sube antes que InfluxDB.

### 6.5 Puerto MQTT accesible desde la red

```bash
nc -zv 192.168.110.165 1883
```

Debe responder **succeeded**.

### 6.6 Datos en InfluxDB

```bash
kubectl exec -n energy-monitoring deploy/influxdb -- \
  influx -username admin -password admin123 -database tasmota \
  -execute 'SHOW MEASUREMENTS'
```

Esperado: `tasmota_energia`

```bash
kubectl exec -n energy-monitoring deploy/influxdb -- \
  influx -username admin -password admin123 -database tasmota \
  -execute 'SELECT time, "ENERGY_Power", "ENERGY_Voltage", "ENERGY_Total" FROM "tasmota_energia" ORDER BY time DESC LIMIT 5'
```

Si ves valores reales del Sonoff, el pipeline completo funciona.

### 6.7 Prueba MQTT manual (sin Tasmota)

```bash
kubectl run mqtt-test --rm -i --restart=Never -n energy-monitoring \
  --image=eclipse-mosquitto -- \
  mosquitto_pub -h mosquitto -p 1883 \
  -t "tele/tasmota_E8D2BC/SENSOR" \
  -m '{"ENERGY":{"Power":123.4,"Voltage":220}}'
```

### 6.8 Grafana accesible

```bash
kubectl port-forward -n energy-monitoring svc/grafana 3000:3000
```

Abre http://localhost:3000 → `admin` / `admin`.

---

## 7. Configuración de Grafana

### 7.1 Acceder a Grafana

En k3d, Grafana no está expuesto por defecto en el puerto 32300. Usa port-forward:

```bash
kubectl port-forward -n energy-monitoring svc/grafana 3000:3000
```

Login: `admin` / `admin` (te pedirá cambiar la contraseña en el primer acceso).

### 7.2 Agregar datasource InfluxDB

1. Menú → **Connections** → **Data sources** → **Add data source** → **InfluxDB**
2. Configuración:

| Campo | Valor |
|-------|-------|
| Query Language | **InfluxQL** (no Flux) |
| URL | `http://influxdb:8086` |
| Database | `tasmota` |
| User | `admin` |
| Password | `admin123` |

3. **Save & test** → debe aparecer "datasource is working".

> Si falla, abre otro port-forward de InfluxDB y usa `http://localhost:8086`:
> `kubectl port-forward -n energy-monitoring svc/influxdb 8086:8086`

### 7.3 Crear dashboard básico — Potencia

1. **Dashboards → New → New dashboard → Add visualization**
2. Datasource: **InfluxDB**
3. Query:

```sql
SELECT mean("ENERGY_Power") FROM "tasmota_energia" WHERE $timeFilter GROUP BY time($__interval) fill(null)
```

4. Título: `Potencia (W)` | Unidad: **Watt (W)**
5. Rango de tiempo: **Last 1 hour**
6. **Save dashboard**

### 7.4 Paneles adicionales sugeridos

| Panel | Query | Unidad Grafana |
|-------|-------|----------------|
| Voltaje | `SELECT mean("ENERGY_Voltage") FROM "tasmota_energia" WHERE $timeFilter GROUP BY time($__interval) fill(null)` | Volt (V) |
| Corriente | `SELECT mean("ENERGY_Current") FROM "tasmota_energia" WHERE $timeFilter GROUP BY time($__interval) fill(null)` | Ampere (A) |
| Consumo total | `SELECT last("ENERGY_Total") FROM "tasmota_energia" WHERE $timeFilter GROUP BY time($__interval) fill(null)` | kilowatt-hour (kWh) |
| Factor de potencia | `SELECT mean("ENERGY_Factor") FROM "tasmota_energia" WHERE $timeFilter GROUP BY time($__interval) fill(null)` | none (0–1) |

---

## 8. Datos recopilados y unidades

Telegraf recibe JSON de Tasmota por MQTT y lo guarda en InfluxDB bajo la measurement **`tasmota_energia`**. Los campos anidados `ENERGY.*` se aplanan como `ENERGY_*`.

### Ejemplo del mensaje MQTT original

```json
{
  "Time": "2026-06-09T13:00:00",
  "ENERGY": {
    "TotalStartTime": "2026-01-01T00:00:00",
    "Total": 3155.053,
    "Yesterday": 4.265,
    "Today": 1.022,
    "Period": 5,
    "Power": 296,
    "ApparentPower": 308,
    "ReactivePower": 83,
    "Factor": 0.96,
    "Voltage": 100,
    "Current": 3.067
  }
}
```

### Campos almacenados en InfluxDB

| Campo InfluxDB | Campo Tasmota | Unidad | Descripción |
|----------------|---------------|--------|-------------|
| `ENERGY_Power` | Power | **W** (vatios) | Potencia activa instantánea consumida por la carga |
| `ENERGY_Voltage` | Voltage | **V** (voltios) | Tensión de línea medida |
| `ENERGY_Current` | Current | **A** (amperios) | Corriente de línea |
| `ENERGY_ApparentPower` | ApparentPower | **VA** (volt-amperios) | Potencia aparente |
| `ENERGY_ReactivePower` | ReactivePower | **var** (volt-amperios reactivos) | Potencia reactiva de la carga |
| `ENERGY_Factor` | Factor | **adimensional** (0–1) | Factor de potencia (coseno φ) |
| `ENERGY_Total` | Total | **kWh** | Consumo acumulado total (incluye hoy) |
| `ENERGY_Today` | Today | **kWh** | Consumo de hoy (desde medianoche) |
| `ENERGY_Yesterday` | Yesterday | **kWh** | Consumo de ayer (00:00–24:00) |
| `ENERGY_Period` | Period | **Wh** (vatios-hora) | Energía consumida desde el mensaje anterior |
| `topic` | (tag) | — | Tópico MQTT origen, ej. `tele/tasmota_E8D2BC/SENSOR` |

> Referencia oficial de unidades ENERGY: [Tasmota — Sonoff Pow R2 (tabla de telemetría)](https://tasmota.github.io/docs/devices/Sonoff-Pow-R2/)

### Frecuencia de recolección

| Componente | Intervalo |
|------------|-----------|
| Tasmota (`TelePeriod`) | 10 s (configurado) o 300 s (default) |
| Telegraf | 10 s (`interval = "10s"` en `telegraf.conf`) |
| InfluxDB | Cada mensaje procesado por Telegraf |

---

## 9. Arquitectura del flujo de datos

```
┌─────────────────────┐     MQTT          ┌──────────────┐
│  Sonoff POWR316D    │  tele/.../SENSOR  │   Mosquitto  │
│  (Tasmota)          │ ────────────────► │   :1883      │
└─────────────────────┘                   └──────┬───────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │   Telegraf   │
                                          └──────┬───────┘
                                                 │ HTTP
                                                 ▼
                                          ┌──────────────┐
                                          │   InfluxDB   │
                                          │ tasmota_     │
                                          │ energia      │
                                          └──────┬───────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │   Grafana    │
                                          │  Dashboards  │
                                          └──────────────┘
```

---

## 10. Credenciales del sistema

| Servicio | Usuario | Contraseña | Notas |
|----------|---------|------------|-------|
| InfluxDB | `admin` | `admin123` | Base de datos: `tasmota` |
| Grafana | `admin` | `admin` | Cambiar en primer login |
| Mosquitto | — | — | Acceso anónimo habilitado |
| Tasmota web | — | — | Sin credenciales por defecto |

---

## 11. Comandos de mantenimiento

```bash
# Ver estado general
kubectl get all -n energy-monitoring

# Logs en tiempo real
kubectl logs -n energy-monitoring -l app=telegraf -f
kubectl logs -n energy-monitoring -l app=mosquitto -f

# Reiniciar un servicio
kubectl rollout restart deployment/telegraf -n energy-monitoring

# Consultar último valor de potencia
kubectl exec -n energy-monitoring deploy/influxdb -- \
  influx -username admin -password admin123 -database tasmota \
  -execute 'SELECT LAST("ENERGY_Power") FROM "tasmota_energia"'

# Eliminar todo el stack
kubectl delete -f kubernetes_deployment

# Eliminar el cluster k3d
k3d cluster delete energy-monitoring
```

---

## 12. Troubleshooting rápido

| Síntoma | Causa probable | Solución |
|---------|----------------|----------|
| `connection refused` en `kubectl get nodes` | Sin cluster k3d | `k3d cluster create energy-monitoring -p "1883:1883@loadbalancer"` |
| `pod does not have a host assigned` | Pod aún arrancando | Esperar `Running 1/1` |
| Tasmota no conecta a MQTT | Puerto 1883 no expuesto | Recrear k3d con `-p "1883:1883@loadbalancer"` |
| `SHOW MEASUREMENTS` vacío | Sin datos MQTT | Revisar topic en Telegraf y logs de Mosquitto |
| Solo datos de prueba (`123.4`) | Tasmota no publica | Verificar Host/Port en Tasmota y conexión en logs Mosquitto |
| Grafana sin datos | Rango de tiempo o datasource | Last 1 hour + InfluxQL + DB `tasmota` |
| Valores de energía incorrectos | Sin calibración | Calibrar en consola Tasmota (`VoltageSet`, `CurrentSet`) |
| Error zsh con comentarios | Paréntesis en comentario `#` | Usar comentarios simples sin `()` |

---

## Despliegue en producción (NUC con K3s)

Para un entorno doméstico permanente en un Intel NUC u otro servidor con K3s:

1. Instalar K3s en el NUC.
2. Copiar kubeconfig al equipo de administración.
3. Aplicar el manifiesto: `kubectl apply -f kubernetes_deployment`
4. Acceder a servicios por IP del NUC:
   - Grafana: `http://<IP_NUC>:32300`
   - InfluxDB: `http://<IP_NUC>:32086`
   - Mosquitto: `mqtt://<IP_NUC>:1883`
5. Configurar Tasmota con la IP del NUC como **MqttHost**.

---

## Licencia

MIT License — ver [README.md](README.md).
