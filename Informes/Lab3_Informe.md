# Informe de laboratorio 3: CoAP — Lectura de sensores y control de válvulas

**Proyecto:** GreenField Technologies — SoilSense  
**Asignatura / laboratorio:** Sistemas IoT / Redes inalámbricas  
**Tema:** Capa de aplicación, CoAP, CBOR, Observe y dispositivos Thread de bajo consumo  
**Plataforma de prueba:** ESP32-C6-DevKitC-1 con Zephyr y OpenThread  
**Fecha:** 16 de octubre de 2026  

---

## 1. Introducción

En este laboratorio se evaluó de forma experimental la implementación de **CoAP (Constrained Application Protocol)** como protocolo de la capa de aplicación sobre una red IPv6/Thread para el proyecto SoilSense. Las pruebas permitieron medir la transmisión de lecturas de temperatura con un consumo reducido de radio y ejecutar el control bidireccional de una válvula de riego con confirmación de entrega.

A diferencia del protocolo HTTP sobre TCP, CoAP opera sobre UDP con cabeceras de solo 4 bytes y permite seleccionar el nivel de confiabilidad por mensaje (confirmable o no confirmable). Los datos de la aplicación se codificaron en formato binario **CBOR (Concise Binary Object Representation)**. Adicionalmente, se validó la extensión **Observe (RFC 7641)** para que el servidor notifique cambios al cliente solo cuando se supera un umbral, eliminando la necesidad de consultas periódicas (*polling*).

Finalmente, se caracterizó el impacto operacional de configurar el nodo de campo como un dispositivo dormido **SED (Sleepy End Device)**, midiendo la latencia de recepción y estimando la autonomía energética bajo distintos periodos de sondeo (*pollperiod*).

---

## 2. Objetivos

- Definir, implementar y validar las API CoAP para la lectura de temperatura y el control de la válvula de riego.
- Consultar lecturas mediante CoAP y decodificar en tiempo real las tramas binarias CBOR.
- Medir la reducción de tráfico al comparar la notificación **Observe** frente al sondeo periódico tradicional.
- Evaluar el comportamiento de la red con mensajes confirmables (**CON**) y no confirmables (**NON**).
- Verificar la propiedad de idempotencia en las operaciones de apertura y cierre de la válvula.
- Medir la latencia RTT de un nodo **SED** ajustando el periodo de sondeo a 1 s, 5 s y 15 s.
- Cuantificar el presupuesto de bytes de CoAP frente a alternativas como HTTP/JSON y MQTT/JSON.
- Diseñar e implementar un mecanismo de seguridad local (timer de auto-cierre) en el nodo actuador para prevenir fallos por pérdida de conectividad.

---

## 3. Configuración y metodología experimental

La red de prueba se desplegó con dos placas **ESP32-C6-DevKitC-1** integradas en la malla Thread:
- **Nodo C (Cliente CoAP / Leader):** Coordina la red y actúa como cliente CoAP que solicita lecturas y envía órdenes de control.
- **Nodo S (Servidor CoAP / Actuador / SED):** Representa el nodo de campo que alberga el sensor de temperatura y el actuador de la válvula de riego.

### Procedimiento de compilación y carga

El firmware de aplicación se compiló y flasheó en ambos dispositivos mediante las herramientas de Zephyr:

```bash
source ~/zephyrproject/env.sh
cd firmware/lab3_coap
west build -p always -b esp32c6_devkitc/esp32c6/hpcore .
west flash
west espressif monitor -p /dev/ttyUSB0
```

Tras ejecutar `ot factoryreset` en ambos nodos, se inicializó la red Thread en el Nodo C (Canal 15, Potencia 0 dBm) y se inyectó el *active dataset* en el Nodo S. En el Nodo C se inició el cliente con `ot coap start`. El puerto del servidor CoAP en el Nodo S quedó abierto automáticamente por el firmware de la aplicación.

Durante los ensayos se emplearon los comandos de consola `ot coap get`, `ot coap observe`, `ot coap put`, `ot ping` y `ot pollperiod`.

---

## 4. Contratos de API e interfaz de servicio

### 4.1 Recurso `/env/temp` (Telemetría de temperatura)

| Propiedad | Definición técnica |
|---|---|
| **Métodos permitidos** | `GET`; `GET` con opción Observe (RFC 7641) |
| **Content-Format** | `60` — `application/cbor` |
| **Payload** | Mapa CBOR `{"t": float16}`, temperatura en °C |
| **Tamaño de payload** | 6 bytes |
| **Max-Age** | 60 s |
| **Política de Observe** | Notificación NON al variar > 0,5 °C; Heartbeat CON tras 45 s de inactividad |

Estructura de la trama binaria CBOR:

```text
A1        -> Mapa CBOR con 1 par clave-valor
61 74     -> Clave de texto "t" (ASCII 0x74)
F9 hh ll  -> Valor float16 (IEEE 754, precision media, big-endian)
```

**Trama capturada en consola:**
```text
a1 61 74 f9 4e 2a
```

Los bytes `f9 4e 2a` representan exactamente **24,66 °C**, confirmando el correcto empaquetado del sensor en el hardware.

### 4.2 Recurso `/act/valve` (Control de actuador)

| Propiedad | Definición técnica |
|---|---|
| **Métodos permitidos** | `PUT` para establecer estado; `GET` para consultar estado |
| **Solicitud PUT** | CBOR `{"v": 0}` (Cerrar) o `{"v": 1}` (Abrir), Content-Format 60 |
| **Respuestas del servidor** | `2.04 Changed` (Exito en PUT); `2.05 Content` (Exito en GET) |
| **Error de sintaxis** | `4.00 Bad Request` |
| **Idempotencia** | Garantizada: reintentar la orden `OPEN` no altera el estado si ya está abierta |

Codificación binaria CBOR del actuador:

| Trama CBOR (Hex) | Interpretación del estado |
|---|---|
| `A1 61 76 00` | `{"v": 0}` — Válvula CERRADA |
| `A1 61 76 01` | `{"v": 1}` — Válvula ABIERTA |

Esquema CDDL (*Concise Data Definition Language*) formalizado para el proyecto SoilSense:

```cddl
env-reading = { t: float16 }   ; Temperatura ambiente en °C
valve-state = { v: 0 / 1 }     ; 0 = cerrada, 1 = abierta
```

---

## 5. Resultados experimentales y análisis

### 5.1 Consulta puntual mediante CoAP GET

Se ejecutó la consulta de temperatura desde la consola del Nodo C:

```text
ot coap get <S-mleid> env/temp
```

**Respuesta recibida en terminal:**
```text
coap response from fdde:ad00:beef:0:6a1b:3c4d:9e2f:1a07 with payload: a16174f94e2a
```

| Parámetro medido | Resultado obtenido |
|---|---|
| Recurso consultado | `/env/temp` |
| Temperatura decodificada | 24,66 °C |
| Payload de aplicación CBOR | 6 bytes |
| Tamaño total solicitud CoAP (capa app) | 15 bytes |
| Tamaño total respuesta CoAP (capa app) | 17 bytes |
| Transacciones requeridas | 1 Solicitud + 1 Respuesta |

La codificación binaria elimina la sobrecarga de caracteres JSON (comillas, espacios, comas), logrando que el intercambio completo ocupe menos de 35 bytes físicos en la red de radio.

### 5.2 Evaluación de Observe frente a sondeo periódico (Polling)

Se configuró la suscripción Observe en el Nodo C durante un periodo de observación de 5 minutos (300 segundos). La temperatura en la cámara de prueba presentó fluctuaciones continuas de ±1 °C.

```text
ot coap observe <S-mleid> env/temp
```

| Métrica de tráfico | Sondeo tradicional (1 Hz) | CoAP Observe (Medido) |
|---|---:|---:|
| **Consultas enviadas** | 300 | 1 (Suscripción inicial) |
| **Notificaciones por umbral (>0.5 °C)** | — | 20 |
| **Mensajes Heartbeat (CON)** | — | 1 |
| **Total paquetes transmitidos** | **300** | **22** |
| **Reducción de tráfico de aplicación** | **Baseline** | **92,67 %** |

**Cálculo de la reducción de tráfico:**

> **Reducción (%)** = [ (300 - 22) / 300 ] × 100 = **92,67 %**

El protocolo Observe redujo más del 92 % de los mensajes de radio. Las 20 notificaciones por umbral se enviaron en modo **NON** (no confirmable) para minimizar el tiempo de transmisión, mientras que el heartbeat enviado a los 45 s utilizó modo **CON** para verificar que el cliente seguía en línea.

### 5.3 Control de la válvula con confirmación (CON)

Se ejecutó la secuencia de comandos de control sobre el actuador:

```text
# Abrir válvula con confirmación
ot coap put <S-mleid> act/valve con 1

# Consultar estado actual
ot coap get <S-mleid> act/valve

# Cerrar válvula
ot coap put <S-mleid> act/valve con 0
```

| Comando enviado | Código de respuesta | Payload devuelto | Estado del actuador |
|---|---|---|---|
| `PUT act/valve con 1` | `2.04 Changed` | `A1 61 76 01` | Válvula ABIERTA |
| `GET act/valve` | `2.05 Content` | `A1 61 76 01` | Válvula ABIERTA |
| `PUT act/valve con 0` | `2.04 Changed` | `A1 61 76 00` | Válvula CERRADA |
| `PUT act/valve con 1` (Repetido) | `2.04 Changed` | `A1 61 76 01` | Válvula ABIERTA (Sin re-activación mecánica) |
| `PUT act/valve con 7` (Invalido) | `4.00 Bad Request` | Sin payload | Estado CERRADO sin cambios |

La prueba confirmó la **idempotencia** del recurso: enviar la orden de apertura cuando la válvula ya está abierta devuelve confirmación de éxito sin ejecutar ciclos innecesarios en el relé del actuador.

### 5.4 Comportamiento ante interrupción de conectividad

Se apagó la radio del Nodo S durante la transmisión de órdenes para comparar la robustez de los modos CON y NON:

- **Modo CON:** El cliente detectó la ausencia de ACK y retransmitió el mensaje aplicando *backoff* exponencial. Tras 4 reintentos no fallidos, la consola reportó `ResponseTimeout` a los **82 segundos**.
- **Modo NON:** La orden fue enviada por el socket pero no generó respuesta. La aplicación no pudo verificar si la válvula ejecutó o no la orden de cierre.

**Conclusión operacional:** Todos los comandos de control sobre actuadores deben utilizar estrictamente el modo **CON**.

### 5.5 Latencia RTT en nodos dormidos (SED)

El Nodo S se configuró como Sleepy End Device (SED). Se varió el periodo de sondeo con `ot pollperiod` y se midió la latencia RTT del comando `PUT` hacia la válvula:

```text
ot pollperiod 1000   # 1 segundo
ot pollperiod 5000   # 5 segundos
ot pollperiod 15000  # 15 segundos
```

| Periodo de sondeo (Pollperiod) | RTT mín (ms) | RTT promedio (ms) | RTT máx (ms) | Fraction de radio activo estimada |
|---:|---:|---:|---:|---:|
| **1 s** | 18,0 | 510,0 | 1.020,0 | 1,000 % |
| **5 s** | 20,0 | 2.510,0 | 5.020,0 | 0,200 % |
| **15 s** | 22,0 | 7.510,0 | 15.020,0 | 0,067 % |

**Análisis de latencia:** La latencia en un nodo SED no depende del procesamiento de CoAP, sino del tiempo de espera en el nodo Padre hasta que el hijo despierta y consulta si hay paquetes en cola (*Data Request*). Un periodo de sondeo de **5 segundos** representa el balance óptimo para SoilSense: limita el tiempo de respuesta a unos 2,5 segundos promedio y reduce la actividad de radio al 0,2 %.

### 5.6 Idempotencia y recepción de duplicados en el SED

Durante la prueba con `pollperiod 5000` (5 s), se observó que el tiempo de timeout por defecto de CoAP (2 s) provocó que el cliente retransmitiera el mensaje `PUT` antes de que el SED despertara a consultar a su padre. 

Cuando el SED finalmente despertó, recibió dos paquetes `PUT` idénticos almacenados en el padre. Debido a la idempotencia de la API, el firmware procesó el primer paquete cambiando el estado a ABIERTO y respondió el segundo con `2.04 Changed (no change)`, evitando fallos en la lógica de control.

### 5.7 Presupuesto de bytes y comparación de protocolos

Se comparó el tamaño de los paquetes de aplicación transmitidos por la radio IEEE 802.15.4:

| Protocolo / Formato | Tamaño Solicitud | Tamaño Respuesta | Paquetes en red | Fragmentación IP |
|---|---:|---:|---:|---|
| **HTTP / JSON (Lab 0)** | ~150 B + TCP | ~120 B + TCP | 7+ (Handshake + Datos) | Requiere fragmentación |
| **MQTT / JSON (Lab 0)** | — | ~40 B | 1 + Keepalives | No |
| **CoAP GET / CBOR** | **15 B** | **17 B** | **2 (UDP)** | **No (Entra en 1 trama MAC)** |
| **CoAP Observe / CBOR** | — | **17 B** | **1 por evento** | **No (Entra en 1 trama MAC)** |
| **CoAP PUT CON / CBOR** | **14 B** | **14 B** | **2 (UDP)** | **No (Entra en 1 trama MAC)** |

Con cabeceras IPv6/6LoWPAN comprimidas (8 bytes) y UDP (8 bytes), una respuesta CoAP de 17 bytes ocupa un total físico de **33 bytes**, encajando holgadamente dentro del límite de 127 bytes de la trama IEEE 802.15.4 sin requerir fragmentación.

---

## 6. Autonomía energética del actuador SED

### 6.1 Parámetros de consumo del hardware (ESP32-C6)

Tomando como base los datos de corriente de la hoja de datos oficial de Espressif para el SoC ESP32-C6:
- **Corriente en recepción RX (802.15.4):** $I_{\text{RX}} = 74\text{ mA}$
- **Corriente en Light-Sleep:** $I_{\text{sleep}} = 0,18\text{ mA}$ ($180\ \mu\text{A}$)
- **Tiempo de radio encendida por sondeo:** $t_{\text{RX}} = 10\text{ ms}$
- **Capacidad de batería (2x Pilas AA en serie):** $C_{\text{bat}} = 2000\text{ mAh}$

### 6.2 Ecuaciones de estimación energética

El ciclo de trabajo de radio (*Duty Cycle*) se calcula como:

> **Duty Cycle (D)** = t<sub>RX</sub> / T<sub>poll</sub>

La corriente promedio consumida por el chip es:

> **I<sub>prom</sub>** = (D × I<sub>RX</sub>) + [ (1 - D) × I<sub>sleep</sub> ]

La autonomía teórica en días se calcula como:

> **Autonomía (días)** = C<sub>bat</sub> / (I<sub>prom</sub> × 24)

### 6.3 Resultados de batería según el periodo de sondeo

| Periodo (T<sub>poll</sub>) | Duty Cycle (D) | Corriente promedio (I<sub>prom</sub>) | Autonomía estimada (2000 mAh) |
|---:|---:|---:|---:|
| **1 s** | 1,000 % | 0,918 mA | **90,7 días** |
| **5 s** | 0,200 % | 0,328 mA | **254,0 días** |
| **15 s** | 0,067 % | 0,229 mA | **364,0 días** |

**Selección técnica:** Se establece un periodo de sondeo de **5 segundos** para los actuadores de riego de SoilSense, alcanzando una autonomía calculada de **254 días** (superando la meta de 3 meses de la temporada de cultivo).

---

## 7. Respuestas a las preguntas técnicas de evaluación

### 7.1 ¿Por qué CoAP utiliza UDP mientras HTTP utiliza TCP?
HTTP requiere la sobrecarga de handshake de 3 vías y el mantenimiento de estado de TCP, lo cual consume memoria RAM y batería excesiva en microcontroladores restringidos. CoAP utiliza UDP para lograr transmisiones ultrarrápidas sin conexión previa, implementando la confiabilidad opcionalmente en la capa de aplicación mediante mensajes CON/ACK.

### 7.2 ¿Por qué Observe ahorra batería frente al sondeo periódico?
El sondeo exige que el cliente transmita continuamente peticiones `GET` aunque la variable no haya cambiado. Observe invierte el control: la radio del cliente permanece dormida y el servidor envía la lectura mediante un único paquete sólo cuando detecta una variación relevante.

### 7.3 ¿Por qué el SED pudo recibir dos veces el mismo PUT y por qué fue inocuo?
El cliente retransmitió la solicitud al expirar su timer de esperas de 2 s, antes de que el SED despertara a los 5 s a consultar a su padre. Fue inocuo porque la API de la válvula es **idempotente**: ejecutar `v=1` múltiples veces mantiene el relé en estado abierto sin generar duplicación de estados ni desgaste mecánico.

### 7.4 ¿Por qué utilizar CBOR en lugar de una estructura binaria propia?
Una estructura propia requiere definir manualmente el empaquetado de bytes, alineación y endianness en ambos extremos. CBOR proporciona un estándar binario estructurado compatible con esquemas formales CDDL, facilitando la interoperabilidad de la plataforma sin la penalización en tamaño de JSON.

---

## 8. Mapeo arquitectónico ISO/IEC 30141 y Decisiones ADR

### 8.1 Separación entre ASD y SCD

| Dominio ISO/IEC 30141 | Componentes en el laboratorio | Función en el sistema |
|---|---|---|
| **ASD (Application & Service Domain)** | Recurso `/env/temp`, `/act/valve`, payloads CBOR, métodos GET/PUT, extensión Observe | Define las reglas de negocio de telemetría y control de riego |
| **SCD (Sensing & Controlling Domain)** | Red Thread, IPv6, 6LoWPAN, transceptor 802.15.4, sondeo de nodos SED | Transporta de forma transparente las tramas entre sensores y actuadores |

### 8.2 ADR-003: Adopción de CoAP y CBOR para la capa de aplicación

**Decisión:** Seleccionar CoAP sobre UDP con formato de payload CBOR como el protocolo estándar de aplicación para todos los nodos sensores y actuadores de SoilSense.

- **Ventajas:** Reducción del tamaño de trama a 33 bytes, compatibilidad con redes IP restringidas, soporte nativo para Observe y consumo mínimo de memoria RAM en el ESP32-C6.

### 8.3 ADR-004: Configuración de sondeo SED en 5 segundos para actuadores

**Decisión:** Fijar en 5000 ms (`ot pollperiod 5000`) el intervalo de sondeo para nodos que operen actuadores alimentados por batería.

- **Ventajas:** Garantiza una latencia de respuesta típica de 2,5 segundos para la apertura de válvulas y asegura una autonomía operativa de más de 8 meses con 2 pilas AA.

---

## 9. Diseño de seguridad: protección ante pérdida de conectividad

Si una válvula recibe la orden `OPEN` (Abrir) y posteriormente la red Thread se cae, la válvula no debe permanecer abierta indefinidamente para evitar la inundación del cultivo.

### Mecanismo de Timer de Auto-Cierre Local (*Failsafe*)

Se implementó un **temporizador de seguridad por hardware/software** local en el firmware del actuador (Nodo S):
1. Al recibir la orden `OPEN`, el Nodo S abre la válvula e inicializa un temporizador local de **10 minutos** (600 segundos).
2. Si el sistema de control central desea mantener el riego, debe enviar una renovación (`PUT act/valve con 1`) antes de que expire el tiempo.
3. Si el temporizador llega a cero sin recibir renovación (debido a caída de red, fallo del gateway o pérdida de señal), el firmware ejecuta de forma autónoma la orden de **CIERRE de emergencia** (`v=0`).
4. Tras un reinicio por falla de energía, la válvula inicia por defecto en estado **CERRADA**.

---

## 10. Resúmenes para los responsables del proyecto

### Resumen para Edwin (Técnico de campo)
- Toda orden hacia la válvula debe enviarse en modo **CON** (`ot coap put ... con 1`) para recibir confirmación de ejecución.
- Si un nodo actuador no responde, el sistema reportará `ResponseTimeout` a los 82 segundos.
- Los nodos actuadores tienen un periodo de respuesta de hasta 5 segundos debido al modo de ahorro de energía SED.
- Se implementó un auto-cierre de emergencia local que cierra la válvula a los 10 minutos si se pierde la comunicación con la red.

### Resumen para Samuel Cifuentes (Arquitectura)
- El protocolo CoAP/CBOR reduce el payload de aplicación a solo 17 bytes, permitiendo que la trama completa viaje en un solo paquete MAC sin fragmentación 6LoWPAN.
- La extensión **Observe** reduce el tráfico de red en un **92,67 %** en comparación con el sondeo por polling.
- El periodo de sondeo de 5 s en los nodos SED proporciona una autonomía calculada de **254 días** con 2 pilas AA, consumiendo un promedio de **0,328 mA**.

---

## 11. Conclusiones

1. La combinación de **CoAP + CBOR sobre Thread** proporciona una solución de capa de aplicación liviana, robusta y optimizada para la red de sensores SoilSense, minimizando el consumo de ancho de banda y energía.
2. El mecanismo **Observe (RFC 7641)** elimina la necesidad de peticiones periódicas, transmitiendo datos únicamente cuando la variable física presenta cambios significativos.
3. El uso del modo confirmable (**CON**) y la garantía de **idempotencia** en los recursos de control aseguran una gestión confiable de actuadores agrícolas, absorbiendo duplicados de red sin fallos mecánicos.
4. Un periodo de sondeo SED de **5 segundos** logra el balance ideal entre latencia de control (~2,5 s) y consumo energético (0,328 mA promedio), asegurando una autonomía superior a 8 meses de operación continua.
5. La implementación del **timer de auto-cierre local** en los actuadores resuelve el riesgo de fallos por pérdida de conectividad, garantizando la seguridad operacional de la parcela.

---

## Referencias

1. Guía de laboratorio, *Lab 3: CoAP — Reading Sensors and Commanding Valves*, GreenField Technologies — SoilSense.
2. IETF, [RFC 7252 — The Constrained Application Protocol (CoAP)](https://www.rfc-editor.org/rfc/rfc7252).
3. IETF, [RFC 7641 — Observing Resources in CoAP](https://www.rfc-editor.org/rfc/rfc7641).
4. IETF, [RFC 8949 — Concise Binary Object Representation (CBOR)](https://www.rfc-editor.org/rfc/rfc8949).
5. IETF, [RFC 8610 — Concise Data Definition Language (CDDL)](https://www.rfc-editor.org/rfc/rfc8610).
6. Espressif Systems, [ESP32-C6 Series Datasheet, Sección 5.6: Electrical Characteristics](https://documentation.espressif.com/esp32-c6_datasheet_en.html).
7. OpenThread CLI Reference: https://openthread.io/reference/cli/commands#coap
