# MeteoStation — Estación Meteorológica IoT con ESP32

Estación meteorológica autónoma basada en un microcontrolador **ESP32 AZ-Delivery WROOM-32** que mide temperatura y humedad, publica los datos en tiempo real en GitHub y sirve un dashboard web accesible desde cualquier dispositivo.

---

## ¿Qué hace el proyecto?

- Lee temperatura y humedad cada minuto con un sensor **DHT11**
- Almacena los datos en **GitHub** en archivos JSON separados por día (`data/data_YYYY-MM-DD.json`)
- Publica un **dashboard web en GitHub Pages** con historial, gráficas, estadísticas y alertas — accesible siempre, aunque el ESP32 esté apagado
- Sirve una **página local en tiempo real** accesible desde la red WiFi local
- Se actualiza **de forma inalámbrica (OTA)** sin necesitar cable USB, con verificación SHA256 y rollback automático
- Mantiene un **sistema de logs** en la memoria flash interna del ESP32, sincronizados con GitHub

---

## Tecnologías utilizadas

| Componente | Tecnología |
|---|---|
| Microcontrolador | ESP32 AZ-Delivery WROOM-32 |
| Sensor | DHT11 (temperatura y humedad) |
| Firmware | Arduino IDE / C++ para ESP32 |
| Almacenamiento | GitHub (API REST) + SPIFFS (flash interna) |
| Dashboard público | GitHub Pages + Chart.js |
| Dashboard local | WebServer HTTP embebido en el ESP32 |
| OTA | Arduino Update + verificación SHA256 |

---

## Estructura del repositorio

```
Esp32/
├── esp32.ino           # Firmware del ESP32 (código fuente completo)
├── esp32.bin           # Firmware compilado para OTA
├── version.txt         # Versión actual del firmware
├── hash.txt            # SHA256 del esp32.bin para verificación OTA
├── log.txt             # Log acumulado de eventos
├── log_pending.txt     # Buffer temporal de logs pendientes
├── index.html          # Dashboard público (GitHub Pages)
└── data/
    ├── index.json              # Lista de fechas disponibles
    └── data_YYYY-MM-DD.json    # Datos del sensor por día
```

---

## Cómo revisar el proyecto rápidamente

### 1. Ver el dashboard en vivo
Accede a la página pública del proyecto:
```
https://aminoxamine.github.io/Esp32
```
Muestra temperatura, humedad, gráficas de las últimas 24h, estadísticas del día y alertas.

### 2. Ver los datos raw del sensor
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/data/data_YYYY-MM-DD.json
```
Sustituye `YYYY-MM-DD` por la fecha que quieras consultar.

### 3. Ver el log de eventos del ESP32
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/log.txt
```

### 4. Ver el código fuente del firmware
El archivo `esp32.ino` contiene el firmware completo dividido en dos bloques:
- **Bloque sistema** (arriba): WiFi, OTA, logs, watchdog, rollback, servidor web
- **Bloque estación** (abajo, tras `// ==================== ESTACION METEOROLOGICA ====================`): sensor, subida de datos, dashboard

---

## Despliegue del firmware

### Requisitos
- Arduino IDE 2.x
- Placa: **ESP32 Dev Module**
- Partition Scheme: `Default 4MB with spiffs (1.2MB APP / 1.5MB SPIFFS)`
- Librerías: `DHT sensor library` (Adafruit), `Adafruit Unified Sensor`

### Primera vez (por USB)
1. Abrir `esp32.ino` en Arduino IDE
2. Conectar el ESP32 por USB
3. Seleccionar el puerto COM correcto
4. Compilar y subir (`Ctrl+U`)
5. Abrir Serial Monitor (115200 baudios) y ejecutar:
   ```
   SET_TOKEN ghp_xxxxxxxxxxxxxxxxxxxx
   ```
   Sustituir por un GitHub Personal Access Token con permisos `repo`.

### Actualizaciones posteriores (OTA, sin USB)
1. Compilar: `Sketch → Export Compiled Binary` → genera `esp32.bin`
2. Obtener el hash SHA256 en PowerShell:
   ```powershell
   (Get-FileHash "esp32.bin" -Algorithm SHA256).Hash.ToLower()
   ```
3. Incrementar el número en `version.txt`
4. Subir a GitHub: `esp32.bin`, `hash.txt` (con el hash en minúsculas) y `version.txt`
5. El ESP32 detecta la nueva versión en la próxima comprobación (máximo 5 minutos) y se actualiza solo

---

## Comandos por Serial Monitor

Con el ESP32 conectado por USB y Serial Monitor abierto a 115200 baudios:

| Comando | Acción |
|---|---|
| `STATUS` | Muestra versión, IP, RAM, temperatura, humedad y estado del token |
| `SET_TOKEN ghp_xxx` | Guarda el GitHub Personal Access Token en la memoria flash |
| `RESET_TOKEN` | Elimina el token guardado |

---

## Hardware — Conexiones

| Componente | Pin ESP32 |
|---|---|
| DHT11 VCC | VIN (5V) |
| DHT11 DATA | GPIO 26 |
| DHT11 GND | GND |

> **Importante:** Alimentar el DHT11 desde VIN (5V), nunca desde un GPIO. Los GPIO del ESP32 solo suministran ~12mA, insuficiente para el sensor bajo carga.

---

## Incidencias conocidas

| Síntoma | Causa | Solución |
|---|---|---|
| Sensor devuelve `nan` | Alimentado desde GPIO en lugar de VIN | Conectar VCC a VIN |
| Sensor devuelve `nan` en nuevo ESP32 | Protoboard con conexiones internas defectuosas | Usar cables Dupont directamente |
| Brownout resets | Cargador USB de baja calidad | Usar cargador de al menos 2A |
| Logs dejan de subirse | Token de GitHub caducado | Ejecutar `SET_TOKEN` por serial |
| OTA loop infinito | Bug del SDK al pasar label de partición | Usar `Update.begin(UPDATE_SIZE_UNKNOWN, U_FLASH)` sin label |
