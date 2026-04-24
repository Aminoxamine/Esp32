# MeteoStation — Estación Meteorológica IoT con ESP32

Estación meteorológica autónoma basada en un microcontrolador **ESP32 AZ-Delivery WROOM-32** que mide temperatura, humedad, presión atmosférica, intensidad lumínica y lluvia, publica los datos en tiempo real en GitHub y sirve un dashboard web accesible desde cualquier dispositivo.

---

## ¿Qué hace el proyecto?

- Lee temperatura y humedad cada minuto con un sensor **DHT11**
- Lee presión atmosférica con un sensor **BMP280** (I2C)
- Lee intensidad lumínica con un sensor **BH1750** (I2C)
- Detecta lluvia con un **sensor de lluvia analógico**
- Guarda las lecturas en un **buffer persistente en SPIFFS** — sobrevive a reinicios y cortes de WiFi
- Almacena los datos en **GitHub** en archivos JSON separados por día (`data/data_YYYY-MM-DD.json`)
- Sube el buffer en **tandas de 20 entradas** por ciclo para evitar problemas de memoria
- Publica un **dashboard web en GitHub Pages** con historial por fecha, gráficas, estadísticas y alertas — accesible siempre, aunque el ESP32 esté apagado
- Sirve una **página local en tiempo real** accesible desde la red WiFi local
- Se actualiza **de forma inalámbrica (OTA)** sin necesitar cable USB, con verificación SHA256 y rollback automático
- Mantiene un **sistema de logs** en la memoria flash interna del ESP32, sincronizados con GitHub

---

## Tecnologías utilizadas

| Componente | Tecnología |
|---|---|
| Microcontrolador | ESP32 AZ-Delivery WROOM-32 |
| Sensor temperatura/humedad | DHT11 |
| Sensor presión | BMP280 (I2C) |
| Sensor luz | BH1750 / GY-302 (I2C) |
| Sensor lluvia | Módulo analógico AO/DO |
| Firmware | Arduino IDE / C++ para ESP32 |
| Almacenamiento local | SPIFFS (buffer persistente + logs) |
| Almacenamiento en la nube | GitHub API REST |
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

## Formato de los datos

Cada entrada en `data_YYYY-MM-DD.json` incluye:

```json
{
  "t": 23.5,
  "h": 62.0,
  "p": 1013.2,
  "l": 450.0,
  "r": 12,
  "rain": false,
  "ts": "2026-04-24T10:30:00"
}
```

| Campo | Descripción |
|---|---|
| `t` | Temperatura en °C (DHT11) |
| `h` | Humedad relativa en % (DHT11) |
| `p` | Presión atmosférica en hPa (BMP280) |
| `l` | Intensidad lumínica en lux (BH1750) |
| `r` | Porcentaje de humedad en sensor de lluvia (0-100) |
| `rain` | true si se detecta lluvia |
| `ts` | Timestamp ISO local (CET/CEST) |

---

## Cómo revisar el proyecto rápidamente

### 1. Ver el dashboard en vivo
Accede a la página pública del proyecto:
```
https://aminoxamine.github.io/Esp32
```
Muestra todos los sensores, gráficas históricas con selector de fecha, estadísticas del día y alertas de temperatura.

### 2. Ver los datos raw del sensor
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/data/data_YYYY-MM-DD.json
```
Sustituye `YYYY-MM-DD` por la fecha que quieras consultar.

### 3. Ver las fechas disponibles
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/data/index.json
```

### 4. Ver el log de eventos del ESP32
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/log.txt
```

### 5. Ver el estado en tiempo real (red local)
```
http://[IP_DEL_ESP32]
```
Muestra temperatura, humedad, presión, luz y estado de lluvia en tiempo real con refresco automático cada 30 segundos.

---

## Despliegue del firmware

### Requisitos
- Arduino IDE 2.x
- Placa: **ESP32 Dev Module**
- Partition Scheme: `Default 4MB with spiffs (1.2MB APP / 1.5MB SPIFFS)`
- Librerías:
  - `DHT sensor library` (Adafruit)
  - `Adafruit Unified Sensor` (Adafruit)
  - `Adafruit BMP280 Library` (Adafruit)
  - `BH1750` (Christopher Laws)

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
| `STATUS` | Muestra versión, IP, RAM, todos los sensores, buffer y estado del token |
| `SET_TOKEN ghp_xxx` | Guarda el GitHub Personal Access Token en la memoria flash |
| `RESET_TOKEN` | Elimina el token guardado |
| `CLEAR_BUFFER` | Vacía el buffer SPIFFS (útil para limpiar datos corruptos) |

---

## Hardware — Conexiones

### DHT11 (temperatura y humedad)
| Pin DHT11 | Pin ESP32 |
|---|---|
| VCC | VIN (5V) |
| DATA | GPIO 26 |
| GND | GND |

> **Importante:** Alimentar desde VIN (5V), nunca desde un GPIO. Usar resistencia pull-up de 10kΩ entre DATA y VCC.

### BMP280 y BH1750 — Bus I2C compartido
| Pin sensor | Pin ESP32 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

> BMP280: CSB → 3V3, SDO → GND (dirección I2C 0x76). BH1750: ADD → GND (dirección I2C 0x23). Ambos sensores comparten el mismo bus I2C.

### Sensor de lluvia
| Pin sensor | Pin ESP32 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| AO | GPIO 34 |
| DO | GPIO 35 |

---

## Sistema de buffer persistente

Cuando el ESP32 no tiene WiFi, las lecturas se guardan automáticamente en el archivo `/pending.json` en la memoria flash interna (SPIFFS). Al recuperar la conexión, las entradas se suben en **tandas de 20** cada 5 minutos hasta vaciar el buffer. El buffer tiene un máximo de 200 entradas (~3h de datos). Si se llena, se eliminan las lecturas más antiguas.

---

## Incidencias conocidas

| Síntoma | Causa | Solución |
|---|---|---|
| DHT11 devuelve `nan` | Alimentado desde GPIO en lugar de VIN | Conectar VCC a VIN (5V) |
| DHT11 devuelve `nan` con hardware nuevo | Protoboard con conexiones internas defectuosas | Usar cables Dupont directamente sin protoboard |
| Brownout resets | Cargador USB de baja calidad | Usar cargador de al menos 2A |
| Logs dejan de subirse | Token de GitHub caducado o revocado | Ejecutar `SET_TOKEN ghp_xxx` por serial |
| OTA loop infinito | Bug del SDK al pasar label de partición | Usar `Update.begin(UPDATE_SIZE_UNKNOWN, U_FLASH)` sin label |
| Buffer no se vacía | JSON demasiado grande para la RAM del ESP32 | El sistema sube de 20 en 20 entradas automáticamente |
| BMP280 no encontrado | Dirección I2C incorrecta | Verificar SDO → GND para dirección 0x76 |
| BH1750 no encontrado | Dirección I2C incorrecta | Verificar ADD → GND para dirección 0x23 |
