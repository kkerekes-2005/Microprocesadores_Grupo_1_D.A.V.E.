# D.A.V.E. - Detección Automática de Vehículos y Emergencias
Proyecto final de Microprocesadores (Tecnun). Sistema sensor-actuador por radio Sub-1 GHz.

- **Grupo de proyecto:** _N_
- **Grupo A (nodo Tx, coche):** _integrantes_
- **Grupo B (nodo Rx, monitor):** _integrantes_
- **Frecuencia asignada:** 863,0 MHz (EasyLink_setFrequency(863000000);)

---

## 1. Qué hace

Un nodo hace de **coche** y otro de **monitor**:

- **Nodo Tx (coche):** lee el joystick (conducción simulada), el acelerómetro y el sensor de luz de la EDUMKII, detecta eventos anómalos y los envía por radio.
- **Nodo Rx (monitor):** recibe los eventos y avisa con LCD, zumbador y LED RGB. Con el RSSI de la radio estima la zona de distancia del coche (cerca / medio / lejos) y detecta si se aleja.

| Modo | Activación | Eventos detectados |
|---|---|---|
| Conducción | Por defecto | Frenazo, acelerón, volantazo, conducción errática, impacto |
| Aparcado | Botón S1 del nodo Tx | Movimiento o inclinación, intrusión (cambio de luz), coche alejándose (RSSI) |

```
Joystick + acelerómetro (ADC) + OPT3001 (I²C)
  → MSP430 Tx: detección + pack
  → UART (eUSCI_A3) → CC1310 Tx ─radio→ CC1310 Rx (+RSSI)
  → UART (eUSCI_A3) → MSP430 Rx: unpack + alarma
  → LCD (SPI), zumbador + RGB (PWM)
```

---

## 2. Proyectos

| Proyecto | Placa | Responsable | Descripción |
|---|---|---|---|
| `msp_tx` | MSP-EXP430FR5994 + BOOSTXL-EDUMKII (nodo Tx) | Grupo A | Muestreo, detección de eventos, empaquetado y envío por UART al CC1310 |
| `cc1310_tx` | LAUNCHXL-CC1310 (nodo Tx) | Grupo A | Puente UART → radio (basado en `rfEasyLinkTx`, nortos) |
| `cc1310_rx` | LAUNCHXL-CC1310 (nodo Rx) | Grupo B | Puente radio → UART; añade el RSSI al final de cada datagrama (basado en `rfEasyLinkRx`, nortos) |
| `msp_rx` | MSP-EXP430FR5994 + BOOSTXL-EDUMKII (nodo Rx) | Grupo B | Recepción, verificación, desempaquetado, alarma, LCD y RSSI |

---

## 3. Requisitos

- Code Composer Studio (probado con CCS _versión_)
- SDK del curso `msp-exp430FR5994-edumkii` (plantilla `edumkiilib_empty`, DriverLib, GRLIB)
- SimpleLink CC13x0 SDK `4_20_02_07`
- Por nodo: 1× MSP-EXP430FR5994, 1× BOOSTXL-EDUMKII, 1× LAUNCHXL-CC1310 con antena, 3 cables hembra-hembra

---

## 4. Compilar y cargar

1. En CCS: **Project → Import CCS Projects…** (CCS 20/21: **File → Import Project(s)**) y seleccionar la carpeta de este repositorio. Importar los cuatro proyectos.
2. Comprobar que `cc1310_tx` y `cc1310_rx` tienen la **misma frecuencia** (llamada a `EasyLink_setFrequency()` tras `EasyLink_init()`).
3. Compilar cada proyecto (**Build Project**). Deben compilar sin errores.
4. Cargar cada uno en su placa con **Debug** y, una vez grabado, **Terminate**: el programa queda en la placa y arranca solo al alimentarla por USB.

Orden recomendado de arranque: CC1310 Rx → MSP430 Rx → CC1310 Tx → MSP430 Tx (el sistema se resincroniza solo si no se respeta).

---

## 5. Cableado MSP430 ↔ CC1310 (igual en ambos nodos)

En el **LAUNCHXL-CC1310**, retirar los jumpers **RXD** y **TXD** del bloque de aislamiento y conectar los cables a los pines del **lado del micro**.

| MSP430FR5994 | Dirección | CC1310 |
|---|---|---|
| P6.0 (UCA3TXD) — conector BoosterPack pin 4 | → | DIO2 (RX, jumper RXD) |
| P6.1 (UCA3RXD) — conector BoosterPack pin 3 | ← | DIO3 (TX, jumper TXD) |
| GND | ↔ | GND |

Cada placa se alimenta por su propio USB; no se comparte alimentación. UART a **115200 baudios, 8N1**.

---

## 6. Contrato del datagrama

Tamaño fijo, empaquetado con `memcpy` campo a campo (`common/datagram.{h,c}`, idéntico en `msp_tx` y `msp_rx`). Little-endian (ambos extremos son MSP430).

| Offset | Tamaño | Campo | Tipo | Notas |
|---|---|---|---|---|
| 0 | 1 | Cabecera | `uint8_t` | `0xA5` |
| 1 | 1 | Secuencia | `uint8_t` | +1 por paquete |
| 2 | 1 | Modo | `uint8_t` | 0 = conducción, 1 = aparcado |
| 3 | 1 | Evento | `uint8_t` | 0 = heartbeat, 1 = frenazo, 2 = acelerón, 3 = volantazo, 4 = errático, 5 = impacto, 6 = movimiento aparcado, 7 = intrusión |
| 4 | 2 | \|a\| × 100 | `int16_t` | g × 100 |
| 6 | 1 | Joystick X | `int8_t` | −100…100 |
| 7 | 1 | Joystick Y | `int8_t` | −100…100 |
| 8 | 2 | Luz | `uint16_t` | lux |
| 10 | 1 | Checksum | `uint8_t` | Suma de bytes 0–9 (mod 256) |
| 11 | 1 | RSSI | `int8_t` | dBm — **solo en el lado Rx**, lo añade `cc1310_rx` (`rxPacket.rssi`) |

- MSP430 Tx → CC1310 Tx: **11 bytes**. Por radio: **11 bytes**. CC1310 Rx → MSP430 Rx: **12 bytes**.
- Heartbeat cada 500 ms; eventos al detectarse (cooldown de 1 s por tipo).

> Cualquier cambio en este formato se hace en los dos proyectos MSP430 a la vez.

---

## 7. Periféricos utilizados

| Periférico | `msp_tx` | `msp_rx` |
|---|---|---|
| CS | DCO 16 MHz (MCLK, SMCLK); ACLK = VLO | Ídem |
| Timer_A | Muestreo cada ~20 ms y heartbeat de 500 ms | Refresco del LCD (500 ms) y timeout del enlace |
| Timer_B0 (PWM) | — | Zumbador + LED RGB (variante `hal/pwm/all`) |
| ADC12_B | Joystick + acelerómetro en secuencia (variante `hal/adc/all`) | — |
| eUSCI_B (I²C) | Sensor de luz OPT3001 | — |
| eUSCI_B (SPI) | — | LCD 128×128 (ST7735) con GRLIB |
| eUSCI_A3 (UART) | Envío al CC1310 | Recepción del CC1310 |
| eUSCI_A0 (UART) | CLI de traza por USB | CLI de traza por USB |
| GPIO | S1: cambio de modo; LEDs del LaunchPad: modo actual | S1: reconocer alarma |
| Radio | CC1310 + EasyLink (`cc1310_tx`) | CC1310 + EasyLink (`cc1310_rx`) |

---

## 8. Estructura de los proyectos MSP430

```
msp_tx/ | msp_rx/
├── main.c            inicialización + bucle principal (ISR corta, trabajo en el main)
├── app/              detection.{h,c} (Tx) / monitor.{h,c} (Rx)
├── common/           datagram.{h,c}
├── hal/              hal_adc, hal_i2c, hal_uart (A0), hal_uart_cc (A3), hal_pwm, hal_lcd, hal_digin, hal_digout
└── drivers/          joystick, kxtc9_2050, opt3001, crystalfontz, buzzer, led, switch
```

---

## 9. Depuración

- **CLI:** abrir PuTTY (Serial, 115200) en el puerto *MSP Application UART* de cada LaunchPad para ver los datagramas enviados (Tx) y recibidos (Rx).
- **Escalera de verificación UART:** loopback P6.0–P6.1 → eco contra `uartecho` en el CC1310 → firmware real.
- **Radio:** con el Tx apagado, el Rx no debe recibir nada; si recibe, revisar la frecuencia.

| Síntoma | Causa probable |
|---|---|
| No llega nada por UART | TX/RX sin cruzar, GND sin conectar o jumpers RXD/TXD aún puestos |
| Llega basura | Baudrate distinto de 115200 o GND flojo |
| El Rx no recibe por radio | Frecuencias distintas o antena mal roscada |
| Checksum incorrecto continuo | Formato del datagrama distinto en Tx y Rx |
