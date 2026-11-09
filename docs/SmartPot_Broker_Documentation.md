<!-- portada
eyebrow: Component Technical Documentation
titulo: SmartPot-Broker-Mosquito (Legacy Stack A)
acento: Broker
subtitulo: Legacy Device Messaging Broker
bajada: Eclipse Mosquitto 2.1 deployment with TLS over private CA, dynamic security plugin, per-device isolation, telemetry topic routing, access control lists, and verification suites.
documento: SmartPot-Broker-Mosquito Documentation
version: 1.0 · September 2026
equipo: SmartPotTech
proyecto: smartpot.app
-->

# SmartPot-Broker-Mosquito (Legacy Stack A)

> [!WARNING]
> **Componente Legado (Stack A)**: Este componente ha sido reemplazado en producción por el broker MQTT canónico en EMQX 5.8 ([`SmartPot-Broker`](https://github.com/SmartPotTech/SmartPot-Broker)). Este documento describe la implementación original en Eclipse Mosquitto 2.1 como registro histórico.

## Document Summary

| Field                          | Value                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Proyecto                       | SmartPot · [smartpot.app](https://smartpot.app)                                                                                                                                                                                                                                                                                                                                                                                                         |
| Componente                     | [SmartPot-Broker](https://github.com/SmartPotTech/SmartPot-Broker)                                                                                                                                                                                                                                                                                                                                                                                      |
| Versión                        | 1.0 · septiembre 2026                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| Alcance                        | Listeners, seguridad dinámica, permisos por cultivo, certificados, tópicos, configuración y pruebas                                                                                                                                                                                                                                                                                                                                                     |
| Platform Documentation | [Technical Documentation](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Technical_Documentation.md), [Project Journey](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Project_Journey.md), [Software Lifecycle](https://github.com/SmartPotTech/.github/blob/main/docs/SmartPot_Software_Lifecycle.md), and [Global Diagrams](https://github.com/SmartPotTech/.github/blob/main/docs/README.md#global-diagrams) |
| Maintenance                  | Generated from `docs/` in this repository via `.github/docs/tools`; updated with every component change                                                                                                                                                                                                                                                                                                                   |

<!-- parte: PART I | Component Overview -->

## 1. Purpose

### In Plain Terms

El broker es el cartero entre los dispositivos y la plataforma. El ESP32 de cada cultivo real, físico o en Wokwi, y el
simulador de los cultivos virtuales publican aquí sus lecturas y reciben sus órdenes; la API escucha y responde. Nadie
entra sin usuario y clave, y cada cultivo solo puede tocar sus propios tópicos.

## 2. Component Architecture

<!-- diagrama: SmartPot_Broker_Global_Component | titulo=SmartPot-Broker Component Architecture | lamina=H -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}, "layout": "elk", "elk": {"nodePlacementStrategy": "BRANDES_KOEPF", "mergeEdges": false, "cycleBreakingStrategy": "GREEDY"}}}%%
flowchart LR
  subgraph broker["SmartPot-Broker-Mosquito · Eclipse Mosquitto 2.1"]
    direction TB
    tls["TLS Listener :8883<br/>server.crt · server.key"]
    ws["WebSocket Listener :9001<br/>path /mqtt"]
    dynsec["Dynamic Security Plugin<br/>ACLs per cropId · JSON policy"]
  end
  api["SmartPot-API-Java"]
  device["ESP32 / Wokwi Device"]
  web["PWA Client"]
  device -->|"MQTTS :8883"| tls
  web -->|"WSS :9001"| ws
  api -->|"dynsec control"| dynsec
  classDef core fill:#067A52,stroke:#0B3D2B,color:#FFFFFF
  classDef leaf fill:#DDF5EA,stroke:#067A52,color:#17261F
  class broker,tls,ws,dynsec leaf
  class api,device,web core
```

| Listener                   | Uso                                 | Publicación en producción                      |
|----------------------------|-------------------------------------|------------------------------------------------|
| `8883` MQTT sobre TLS 1.2+ | Dispositivos de los cultivos reales | Directo en `mqtt.smartpot.app:8883`            |
| `9001` WebSocket           | Clientes web                        | `wss://mqtt.smartpot.app/mqtt` detrás de nginx |
| `1883` MQTT                | La API y el simulador               | Solo la red interna de Docker                  |

<!-- parte: PART II | Security & Access Control -->

## 3. Per-Crop Tenant Isolation & ACLs

<!-- diagrama: SmartPot_Broker_01_Device_Permissions | titulo=Qué puede hacer cada conexión -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
sequenceDiagram
  autonumber
  participant API as SmartPot-API-Java
  participant M as Mosquitto Dynamic Security Plugin
  participant D as ESP32 Device
  API->>M: dynsec/createClient (username=cropId, password=deviceKey)
  API->>M: dynsec/addClientRole (scope topic smartpot/v1/{cropId}/#)
  D->>M: MQTT CONNECT (TLS port 8883)
  M-->>D: CONNACK 0 (Authorized & scoped to cropId)
```

La API es la única cuenta administradora: al arrancar crea el rol `device` y vuelve a crear la cuenta de cada cultivo
desde la base de datos (el broker no necesita respaldo propio); al crear, rotar o borrar un cultivo actualiza su cuenta.
Los cultivos virtuales también tienen cuenta: la usa el simulador.

## 4. TLS & WebSocket Certificates

<!-- diagrama: SmartPot_Broker_02_Certificates | titulo=Certificados del broker -->

```mermaid
%%{init: {"theme": "base", "fontFamily": "Segoe UI, Arial, sans-serif", "themeVariables": {"fontFamily": "Segoe UI, Arial, sans-serif", "fontSize": "15px", "primaryColor": "#DDF5EA", "primaryTextColor": "#17261F", "primaryBorderColor": "#067A52", "secondaryColor": "#E3F2FB", "secondaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "secondaryBorderColor": "#1F6FA0", "tertiaryColor": "#F2F7F4", "tertiaryTextColor": "#17261F", "tertiaryBorderColor": "#D5E3DC", "lineColor": "#5B6B63", "textColor": "#17261F", "mainBkg": "#DDF5EA", "nodeBorder": "#067A52", "clusterBkg": "#F7FAF8", "clusterBorder": "#D5E3DC", "edgeLabelBackground": "#FFFFFF", "actorBkg": "#067A52", "actorBorder": "#0B3D2B", "actorTextColor": "#FFFFFF", "actorLineColor": "#5B6B63", "signalColor": "#17261F", "signalTextColor": "#17261F", "labelBoxBkgColor": "#0B3D2B", "labelBoxBorderColor": "#0B3D2B", "labelTextColor": "#FFFFFF", "loopTextColor": "#0B3D2B", "noteBkgColor": "#FDF4DD", "noteBorderColor": "#C98D12", "noteTextColor": "#17261F", "activationBkgColor": "#DDF5EA", "activationBorderColor": "#067A52", "attributeBackgroundColorOdd": "#FFFFFF", "attributeBackgroundColorEven": "#F2F7F4"}}}%%
flowchart LR
  ca["Internal CA Certificate<br/>ca.crt (RSA 4096)"]
  server["Server Certificate<br/>server.crt & server.key"]
  mosq["Mosquitto Daemon<br/>TLS Listener :8883"]
  device["Device Firmware<br/>Embedded ca.crt verification"]
  ca --> server
  server --> mosq
  ca --> device
  device <-->|"TLS 1.3 / 1.2 Handshake"| mosq
  classDef core fill:#067A52,stroke:#0B3D2B,color:#FFFFFF
  classDef leaf fill:#DDF5EA,stroke:#067A52,color:#17261F
  class ca,server core
  class mosq,device leaf
```

El listener TLS solo se habilita si existen `ca.crt`, `server.crt` y `server.key`. `require_certificate` está en
`false`: el dispositivo se autentica con usuario y clave y verifica el servidor con `ca.crt`. Renueva el certificado del
servidor antes de 825 días con la misma CA.

## 5. Tópicos del contrato v1

| Tópico                              | Sentido                                 | QoS |
|-------------------------------------|-----------------------------------------|-----|
| `smartpot/v1/{cropId}/telemetry`    | Dispositivo → API                       | 0   |
| `smartpot/v1/{cropId}/commands`     | API → dispositivo                       | 1   |
| `smartpot/v1/{cropId}/commands/ack` | Dispositivo → API                       | 1   |
| `smartpot/v1/{cropId}/status`       | Dispositivo, retenido y última voluntad | 1   |

<!-- parte: PART III | Operations & Verification -->

## 6. Configuración

| Variable                                     | Uso                                                                 |
|----------------------------------------------|---------------------------------------------------------------------|
| `MQTT_ADMIN_USERNAME`, `MQTT_ADMIN_PASSWORD` | Cuenta administradora; la clave solo se aplica con el volumen vacío |
| `MQTT_TLS_MAX_CONNECTIONS`                   | Conexiones simultáneas por listener (200 por defecto)               |

Reglas fijas: client id vacío rechazado, sesiones persistentes que vencen una hora después de desconectarse y sin acceso
anónimo.

## 7. Pruebas

`sh tests/smoke.sh smartpot-broker:ci` genera una CA desechable y verifica que la telemetría propia llega, que se
descartan publicaciones y suscripciones a nombre de otro cultivo, que se rechazan claves incorrectas, ids vacíos y
anónimos, y que el listener TLS solo acepta credenciales válidas (10 comprobaciones).

## 8. Operación

| Tarea                              | Cómo                                                                                                                           |
|------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Imagen                             | `ghcr.io/smartpottech/smartpot-broker`: usuario `1883`, solo lectura con `tmpfs` y volumen en `/mosquitto/data`, `HEALTHCHECK` |
| Cambiar la clave del administrador | `mosquitto_ctrl dynsec setClientPassword` o recrear el volumen (la API reaprovisiona los cultivos)                             |
| Despliegue                         | Cada cambio en `main` pasa por el CI, publica la imagen y pide el despliegue central de `.github`                              |
