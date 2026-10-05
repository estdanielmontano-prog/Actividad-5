# Control en tiempo real de un brazo robótico con pinza (ESP32 + Python)

Práctica del repositorio **U_Militar**: a partir del archivo `brazo.urdf` (brazo de 2 articulaciones con pinza) se controla el robot simulado usando **4 potenciómetros** conectados a un **ESP32**, que envía las lecturas por **UART** a un script en **Python** que mueve las articulaciones en tiempo real.

---

## 1. Objetivo

- Programar el ESP32 para leer sensores (potenciómetros) y enviar los datos.
- Desarrollar un script Python que reciba los datos por UART y controle el robot.
- Probar el movimiento de las articulaciones y la apertura/cierre de la pinza.
- Validar la comunicación en tiempo real.

## 2. Descripción del robot (`brazo.urdf`)

| Articulación | Tipo | Función | Rango (visor) |
|---|---|---|---|
| `joint_1` | Revoluta | Giro de la base | −143° a 143° |
| `joint_2` | Revoluta | Giro del brazo | −115° a 115° |
| `joint_gripper` | Prismática | Movimiento vertical de la parte roja (soporta los dedos) | 0 a 0.15 |
| `joint_dedo_izq` | Prismática | Dedo izquierdo | 0 a 0.05 |
| `joint_dedo_der` | Prismática | Dedo derecho | 0 a 0.05 |

Materiales: base gris oscuro, brazo 1 azul, brazo 2 naranja, pinza roja, dedos blanco/gris y efector verde.

## 3. Hardware

- 1 × ESP32 DevKit (esp32dev)
- 4 × potenciómetros lineales (10 kΩ)
- Protoboard y cables Dupont
- Cable USB (alimentación + comunicación serial con el PC)

### Conexiones

Cada potenciómetro: pin extremo 1 → **3V3**, pin extremo 2 → **GND**, pin central (cursor) → pin ADC.

| Potenciómetro | Pin ESP32 | Controla |
|---|---|---|
| P1 | GPIO34 | `joint_2` — gira el **brazo** |
| P2 | GPIO35 | `joint_1` — gira la **base** |
| P3 | GPIO32 | `joint_gripper` — mueve **verticalmente la parte roja** (junto con los dedos) |
| P4 | GPIO33 | `joint_dedo_izq` + `joint_dedo_der` — abre y cierra los **dedos** |

> Se usan pines del **ADC1** porque el ADC2 no funciona de forma fiable con WiFi activo. Alimentar los potenciómetros con **3.3 V**, nunca 5 V.

📷 *Coloca aquí la foto del montaje:* `media/montaje.jpg`

## 4. Arquitectura

```
Potenciómetros ──► ESP32 (ADC 12 bits) ──UART/USB 115200──► Python ──► Simulación del URDF
   (P1..P4)         promedio + trama         "P1,P2,P3,P4\n"     mapeo + filtro     (PyBullet)
```

**Protocolo:** una línea de texto cada 20 ms (50 Hz), con valores crudos del ADC (0–4095):

```
2048,1023,4095,0
```

## 5. Estructura del proyecto

```
U_Militar_Brazo/
├── firmware/
│   ├── platformio.ini
│   └── src/main.cpp          # Código del ESP32
├── python/
│   ├── control_robot.py      # Recibe UART y controla el robot
│   └── requirements.txt
├── urdf/
│   └── brazo.urdf            # Copiar desde el repositorio
├── media/                    # Evidencias
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Firmware ESP32 (`firmware/src/main.cpp`)
1. Configura el ADC a 12 bits y atenuación de 11 dB (rango ≈ 0–3.3 V).
2. Cada 20 ms lee los 4 pines promediando 8 muestras para reducir el ruido.
3. Envía la trama `P1,P2,P3,P4\n` por el puerto serie a 115200 baudios.

### 6.2 Script Python (`python/control_robot.py`)
1. **Carga el URDF** en PyBullet y obtiene automáticamente los límites de cada articulación.
2. **Lee la UART** y descarta tramas incompletas o corruptas; siempre usa la última trama disponible para evitar retardo acumulado.
3. **Filtra** la señal con un filtro exponencial (`ALPHA = 0.25`) para eliminar temblor del potenciómetro.
4. **Mapea** `0–4095` → `[límite_inferior, límite_superior]` de cada articulación.
5. **Mueve** el robot con control de posición (`POSITION_CONTROL`) a 240 Hz.
6. Avisa si se pierde la comunicación por más de 1 s.

## 7. Evidencias

🎥 **Video de funcionamiento:** *(https://drive.google.com/drive/u/0/folders/1ABYW4ooAyx6Ivvj62gKILjlh1gHZZKSv)*
