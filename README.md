# MeteoStation — Estación meteorológica con ESP32

Este proyecto es una estación meteorológica hecha con un **ESP32 AZ-Delivery WROOM-32**. Mide temperatura, humedad, presión, luz y lluvia, y sube los datos a GitHub para verlos desde cualquier sitio.

---

## ¿Qué hace?

- Lee los sensores cada minuto (temperatura, humedad, presión, luz y lluvia)
- Guarda las lecturas en la memoria del ESP32 si no hay WiFi
- Sube los datos a GitHub cada 2 minutos en archivos separados por día
- Tiene un **dashboard en GitHub Pages** donde puedes ver las gráficas e historial aunque el ESP32 esté apagado
- Tiene una **página web local** para ver los datos en tiempo real desde tu red WiFi
- Se actualiza solo por WiFi (**OTA**) sin necesitar cable USB
- Guarda un registro de eventos (logs) en la memoria del ESP32 y los sube a GitHub

---

## Componentes usados

| Componente | Para qué sirve |
|---|---|
| ESP32 AZ-Delivery WROOM-32 | El cerebro del proyecto |
| DHT11 | Mide temperatura y humedad |
| BMP280 (I2C) | Mide presión atmosférica |
| BH1750 / GY-302 (I2C) | Mide la intensidad de luz |
| Módulo sensor de lluvia | Detecta si está lloviendo |
| Arduino IDE / C++ | Para programar el ESP32 |
| SPIFFS | Guarda datos en la memoria flash del ESP32 |
| GitHub API | Sube los datos a la nube |
| GitHub Pages + Chart.js | El dashboard público |

---

## Estructura del repositorio

```
Esp32/
├── esp32.ino           # Código del ESP32
├── esp32.bin           # Firmware compilado (para actualizar por OTA)
├── version.txt         # Número de versión actual
├── hash.txt            # SHA256 del esp32.bin (seguridad)
├── log.txt             # Registro de eventos
├── log_pending.txt     # Logs pendientes de subir
├── index.html          # Dashboard de GitHub Pages
└── data/
    ├── index.json              # Lista de fechas con datos
    └── data_YYYY-MM-DD.json    # Datos de cada día
```

---

## Formato de los datos

Cada lectura guardada en `data_YYYY-MM-DD.json` tiene este formato:

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

| Campo | Qué es |
|---|---|
| `t` | Temperatura en °C |
| `h` | Humedad en % |
| `p` | Presión en hPa |
| `l` | Luz en lux |
| `r` | Humedad en el sensor de lluvia (0-100%) |
| `rain` | true si está lloviendo |
| `ts` | Fecha y hora de la lectura |

---

## Ver el proyecto

### Dashboard público
```
https://aminoxamine.github.io/Esp32
```
Aquí puedes ver las gráficas, cambiar de fecha y ver las estadísticas del día.

### Datos de un día concreto
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/data/data_YYYY-MM-DD.json
```
Cambia `YYYY-MM-DD` por la fecha que quieras.

### Fechas disponibles
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/data/index.json
```

### Logs del sistema
```
https://raw.githubusercontent.com/Aminoxamine/Esp32/main/log.txt
```

### Página local en tiempo real
```
http://[IP_DEL_ESP32]
```
Se ve la temperatura, humedad, presión, luz y lluvia actualizándose cada 30 segundos.

---

## Cómo instalar el firmware

### Lo que necesitas
- Arduino IDE 2.x
- Placa: **ESP32 Dev Module**
- Partition Scheme: `Default 4MB with spiffs`
- Librerías (instalar desde Manage Libraries):
  - `DHT sensor library` (Adafruit)
  - `Adafruit Unified Sensor` (Adafruit)
  - `Adafruit BMP280 Library` (Adafruit)
  - `BH1750` (Christopher Laws)

### Primera vez (con cable USB)
1. Abrir `esp32.ino` en Arduino IDE
2. Conectar el ESP32 por USB
3. Seleccionar el puerto COM correcto
4. Compilar y subir con `Ctrl+U`
5. Abrir el Serial Monitor a 115200 baudios y escribir:
   ```
   SET_TOKEN ghp_xxxxxxxxxxxxxxxxxxxx
   ```
   Pon aquí tu GitHub Personal Access Token con permisos `repo`.

### Actualizar sin cable (OTA)
1. Compilar: `Sketch → Export Compiled Binary`
2. Calcular el SHA256 en PowerShell:
   ```powershell
   (Get-FileHash "esp32.bin" -Algorithm SHA256).Hash.ToLower()
   ```
3. Subir a GitHub los tres archivos: `esp32.bin`, `hash.txt` y `version.txt` (con el número incrementado)
4. El ESP32 se actualiza solo en los próximos 2 minutos

---

## Comandos del Serial Monitor

| Comando | Qué hace |
|---|---|
| `STATUS` | Muestra versión, IP, RAM, sensores y estado del token |
| `SET_TOKEN ghp_xxx` | Guarda el token de GitHub |
| `RESET_TOKEN` | Borra el token guardado |
| `CLEAR_BUFFER` | Vacía el buffer de datos pendientes |

---

## Conexiones del hardware

### DHT11
| Pin DHT11 | Pin ESP32 |
|---|---|
| VCC | VIN (5V) |
| DATA | GPIO 26 |
| GND | GND |

> El DHT11 va a **VIN (5V)**, no a 3V3. Usar resistencia de 10kΩ entre DATA y VCC.

### BMP280 y BH1750 (comparten el bus I2C)
| Pin sensor | Pin ESP32 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| SDA | GPIO 21 |
| SCL | GPIO 22 |

> BMP280: CSB → 3V3, SDO → GND (dirección 0x77). BH1750: ADD → GND (dirección 0x23).

### Sensor de lluvia
| Pin módulo | Pin ESP32 |
|---|---|
| VCC | 3V3 |
| GND | GND |
| AO | GPIO 34 |
| DO | GPIO 35 |

---

## Cómo funciona el buffer

Si el ESP32 no tiene WiFi, guarda las lecturas en `/pending.json` en su memoria flash. Cuando vuelve la conexión, sube los datos en tandas de 20 entradas cada 2 minutos hasta vaciarlo. El buffer aguanta hasta 200 entradas (unas 3 horas de datos). Si se llena, borra las más antiguas para hacer sitio.

---

## Problemas conocidos y soluciones

| Síntoma | Causa | Solución |
|---|---|---|
| DHT11 da `nan` | VCC conectado a un GPIO en vez de VIN | Conectar VCC a VIN (5V) |
| DHT11 da `nan` con sensor nuevo | Protoboard defectuosa | Usar cables Dupont directamente |
| ESP32 se reinicia constantemente | Cargador USB de poca potencia | Usar uno de al menos 2A |
| Los logs no se suben | Token de GitHub caducado | Ejecutar `SET_TOKEN` por serial |
| OTA en bucle infinito | Bug del SDK de Arduino para ESP32 | Usar `Update.begin(UPDATE_SIZE_UNKNOWN, U_FLASH)` |
| Buffer no se vacía | JSON demasiado grande | El sistema ya sube de 20 en 20 automáticamente |
| BMP280 no detectado | Dirección I2C incorrecta | Probar con `bmp.begin(0x77)` |
| BH1750 no detectado | Dirección I2C incorrecta | Verificar que ADD está conectado a GND |
