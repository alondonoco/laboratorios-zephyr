# Informe de laboratorio 2: Thread Mesh — direccionamiento, múltiples saltos y autorrecuperación

**Proyecto:** GreenField Technologies — SoilSense  
**Área:** Conectividad de red  
**Tecnología:** Thread sobre IEEE 802.15.4, IPv6 y 6LoWPAN  
**Plataforma de prueba:** ESP32-C6-DevKitC-1  
**Canal de operación:** 15  
**Fecha:** 16 de octubre de 2026  

---

## 1. Objetivos

- Formar una red Thread con tres nodos: A como líder, B como router intermedio y C como nodo de destino.
- Identificar y capturar las direcciones IPv6 reales que utiliza Thread, distinguiendo su alcance y función en el hardware.
- Medir y comparar el tiempo de ida y vuelta (RTT) de paquetes transmitidos en uno y dos saltos.
- Evaluar experimentalmente la capacidad de autorrecuperación de rutas al desconectar el router intermedio.
- Relacionar los datos medidos con la arquitectura ISO/IEC 30141 y establecer recomendaciones técnicas para el despliegue de la red de sensores de campo.

---

## 2. Fundamento teórico

Thread es una tecnología de red mallada de baja potencia que funciona sobre la capa física e IEEE 802.15.4 MAC. A diferencia de una topología en estrella —donde los nodos dependen de una conexión directa con el coordinador o gateway—, Thread permite que los routers reenvíen paquetes para otros dispositivos. De esta forma, un nodo puede comunicarse con otro a través de múltiples saltos de radio.

La capa de adaptación 6LoWPAN permite transportar IPv6 sobre tramas IEEE 802.15.4, cuyo tamaño máximo de payload físico es de 127 bytes. Para ello, comprime campos de la cabecera IPv6 mediante IPHC (IP Header Compression) y fragmenta datagramas cuando el tamaño lo requiere. El protocolo MLE (Mesh Link Establishment) descubre vecinos, mantiene información de la calidad del enlace y participa dinámicamente en la selección de rutas.

La calidad de enlace se representa mediante el indicador LQI / Link Quality (LQ) en una escala de 0 a 3:
- Enlaces con margen de señal superior a 20 dB se clasifican como **LQ 3** (Costo = 1).
- Enlaces con margen superior a 10 dB se clasifican como **LQ 2** (Costo = 2).
- Enlaces con margen superior a 2 dB se clasifican como **LQ 1** (Costo = 4).

El costo total de una ruta corresponde a la suma acumulada de los costos de sus enlaces individuales. Por esta razón, el algoritmo de enrutamiento de Thread prefiere una ruta de dos saltos de alta calidad (Costo total = 1 + 1 = 2) frente a un enlace directo degradado de baja calidad (Costo = 4).

---

## 3. Procedimiento y configuración de la red

La red experimental se desplegó utilizando tres placas ESP32-C6-DevKitC-1 asignadas con las siguientes funciones iniciales:

| Nodo | Función inicial | Papel en las pruebas |
|---|---|---|
| **Nodo A** | Líder (Leader) | Coordina la red y origina el tráfico de ping |
| **Nodo B** | Router | Reenvía paquetes entre A y C; se desconecta en la prueba de autorrecuperación |
| **Nodo C** | Router | Nodo destino de las pruebas de comunicación |

Las pruebas se ejecutaron fijando el canal de operación en 15, configurando la potencia de transmisión en 0 dBm en los tres dispositivos y compartiendo el mismo *active dataset* de Thread. 

Primero se ejecutó un restablecimiento de fábrica en cada placa; posteriormente se inicialización la red desde el Nodo A y se unieron los Nodos B y C mediante la inyección del conjunto de datos activo.

Comandos ejecutados en los dispositivos durante la práctica:

```bash
# Ejecutado en todos los nodos (A, B y C)
ot txpower 0
ot factoryreset

# En Nodo A: creación y arranque de la red Thread
ot dataset init new
ot dataset channel 15
ot dataset commit active
ot ifconfig up
ot thread start
ot state
ot dataset active -x

# En Nodos B y C: configuración e integración a la red
ot dataset set active <DATASET_HEX_DE_A>
ot ifconfig up
ot thread start
ot state

# Verificación de la topología desde la consola del Nodo A
ot router table
```

La consulta de `ot router table` permitió verificar los identificadores de router, direcciones RLOC16, siguiente salto, costo de ruta y calidad de enlace. Para forzar experimentalmente el tráfico a través de dos saltos, se degradó administrativamente la calidad del enlace directo A–C mediante el comando `ot macfilter rss add-lqi`, conservando el enlace disponible pero asignándole una calidad LQ 1. Con esto se verificó cómo el protocolo selecciona la ruta A–B–C.

---

## 4. Resultados y análisis

### 4.1 Formación de la malla y tabla de routers

Durante la prueba de formación de la red, el Nodo A asumió el rol de Líder, mientras que B y C se establecieron como Routers activos. Con los tres equipos en línea de vista, los enlaces iniciales registraron calidad máxima (LQ 3). Al aplicar el filtro MAC para degradar el enlace A–C, la tabla de enrutamiento cambió la preferencia hacia el Nodo B como siguiente salto hacia C.

| Condición del ensayo | Calidad enlace A–B | Calidad enlace B–C | Calidad asignada A–C | Ruta seleccionada por Nodo A hacia C | Costo total de la ruta |
|---|---:|---:|---:|---|---:|
| **Antes de aplicar filtros** | LQ 3 | LQ 3 | LQ 3 | A → C (Directa) | Directa: 1; vía B: 2 |
| **Después de aplicar filtros** | LQ 3 | LQ 3 | LQ 1 | A → B → C (2 saltos) | Directa: 4; vía B: 1 + 1 = 2 |

El resultado medido demuestra que el algoritmo de enrutamiento dinámico selecciona efectivamente la ruta de dos saltos cuando el costo acumulado de los enlaces intermedios (1 + 1 = 2) es inferior al costo del enlace directo (4).

### 4.2 Clasificación de direcciones Thread registradas

A continuación se presentan los tipos de direcciones unicast y multicast IPv6 asignadas y capturadas en los nodos mediante los comandos `ot ipaddr`, `ot ipaddr mleid`, `ot ipaddr rloc`, `ot rloc16` y `ot extaddr`:

| Tipo de dirección | Valor registrado en ensayo | Alcance | Función operacional |
|---|---|---|---|
| **Link-local** | `fe80::8c21:c4a0:7f3b:5d12` | Un enlace de radio | Comunicación local de un solo salto y mensajes de control MLE entre vecinos |
| **RLOC** | `fdde:ad00:beef:0:0:ff:fe00:5800` | Malla Thread | Enrutamiento dinámico y localización del nodo según la topología lógica |
| **ALOC** | `fdde:ad00:beef:0:0:ff:fe00:fc00` | Malla Thread | Identifica un rol de red específico (`fc00` corresponde al rol de Líder) |
| **ML-EID** | `fdde:ad00:beef:0:6a1b:3c4d:9e2f:1a07` | Malla Thread | Identidad fija del dispositivo para el tráfico de la aplicación SoilSense |
| **Multicast link-local** | `ff02::1`, `ff02::2` | Enlace local | Dirección para todos los nodos y routers del enlace local, respectivamente |
| **Multicast mesh-local** | `ff03::1`, `ff03::2` | Malla completa | Dirección para todos los nodos y routers pertenecientes a la malla |

### 4.3 Derivación de direcciones y RLOC16

**Derivación de la dirección link-local:** Se verificó la construcción de la dirección link-local a partir del identificador de interfaz derivado de la dirección MAC extendida (EUI-64). Al comparar la salida de `ot extaddr` con la dirección `fe80::` obtenida con `ot ipaddr`, se confirmó la inversión del bit U/L (bit 1 del primer byte del identificador) requerida por el estándar.

**Derivación del Router ID:** Para la dirección RLOC16 capturada en la prueba (`0x5800`), se calculó el identificador del router desplazando 10 bits a la derecha:

> **Router ID** = `0x5800` >> 10 = **22** (decimal)

El valor obtenido de 22 coincidió con la columna `Router ID` correspondiente en la tabla de routers de `ot router table`. Esto confirma que la dirección RLOC representa la ubicación lógica en la topología (y por tanto cambia si el dispositivo se reasocia), mientras que la dirección ML-EID permanece constante.

### 4.4 Medición del RTT en uno y dos saltos

Para evaluar la latencia introducida por el enrutamiento multisalto, se enviaron ráfagas de 20 pings ICMPv6 de 64 bytes con un intervalo de 0,5 segundos entre transmisiones. Se comparó la comunicación directa de un salto contra la comunicación de dos saltos, incorporando como referencia los datos base obtenidos en el Laboratorio 1.

| Trayecto del ping | Saltos | RTT mín (ms) | RTT prom (ms) | RTT máx (ms) | Paquetes perdidos | Tasa de pérdida |
|---|---:|---:|---:|---:|---:|---:|
| **Nodo A → Nodo B** | 1 | 11,0 | 17,2 | 29,0 | 0 de 20 | 0 % |
| **Nodo A → Nodo C (vía B)** | 2 | 22,0 | 34,6 | 61,0 | 1 de 20 | 5 % |
| **Base Lab 1 (A → B)** | 1 | 9,0 | 15,8 | 27,0 | 0 de 20 | 0 % |

Los datos experimentales muestran un incremento en el RTT promedio de **17,2 ms a 34,6 ms** al añadir un salto intermedio. El incremento neto de **17,4 ms** corresponde al tiempo de procesamiento en el Nodo B, la gestión de colisiones en la capa MAC y el reenvío de tramas a través del medio compartido. La pérdida puntual de 1 paquete en la ruta de dos saltos (5 %) ocurrió durante el primer ping al ML-EID mientras la red resolvía la tabla de mapeo de direcciones.

### 4.5 Prueba de autorrecuperación tras la desconexión del Router B

Para evaluar la tolerancia a fallos, se inició una transmisión continua de pings (1 paquete/segundo) desde el Nodo A hacia la dirección ML-EID del Nodo C a través de la ruta de dos saltos (A–B–C). Con la transmisión en curso, se desconectó físicamente la alimentación del Nodo B. Se registraron los pings perdidos de forma consecutiva hasta la reanudación del tráfico por el enlace de respaldo directo (A–C). La prueba se repitió en 3 ejecuciones independientes:

| Ejecución | Pings perdidos consecutivos | Tiempo de recuperación (s) | Ruta establecida tras la recuperación |
|---:|---:|---:|---|
| **Prueba 1** | 8 | 8,0 | A → C directa (Siguiente salto RLOC16: `0x3F00` / ID: 63) |
| **Prueba 2** | 11 | 11,0 | A → C directa (Siguiente salto RLOC16: `0x3F00` / ID: 63) |
| **Prueba 3** | 7 | 7,0 | A → C directa (Siguiente salto RLOC16: `0x3F00` / ID: 63) |
| **Promedio** | **8,7** | **8,7** | **Ruta directa restaurada por enlace de respaldo** |

**Análisis de reconfiguración:** El tiempo promedio de autorrecuperación fue de **8,7 segundos** (con un máximo de 11,0 segundos). Este valor cumple holgadamente con el requisito operacional del proyecto SoilSense, que establece un límite máximo de recuperación de 120 segundos. 

Al desconectarse el Nodo B, la ausencia de respuestas ACK en la capa MAC desencadenó la actualización de la tabla de vecinos en el Nodo A, conmutando el tráfico hacia el enlace directo de respaldo A–C. Posteriormente, al reconectar el Nodo B y ejecutar `ot thread start`, este se reincorporó como router y la red volvió a enrutar el tráfico vía A–B–C debido a su menor costo.

---

## 5. Respuestas a las preguntas de análisis técnico

### 5.1 ¿Cómo cabe una cabecera IPv6 de 40 bytes en una trama IEEE 802.15.4 de 127 bytes?

6LoWPAN utiliza compresión de cabeceras IPHC (RFC 6282). Dado que campos como la versión de IP, la longitud del payload y los prefijos de red son conocidos de antemano por los nodos de la malla, IPHC los omite o los reemplaza por identificadores cortos de 1 a 2 bytes. Además, las direcciones IPv6 se derivan directamente de las direcciones MAC del emisor y receptor. En los casos donde la cabecera comprimida y los datos superan los 127 bytes físicos, 6LoWPAN aplica fragmentación transparente en la capa de adaptación.

### 5.2 ¿Por qué la recuperación de la comunicación tarda segundos y no minutos?

Aunque la entrada de un router inactivo pueda permanecer hasta 100 segundos en la tabla de routers antes de eliminarse por completo, el desvío del tráfico es mucho más rápido. Cuando el Nodo A intenta transmitir a través de B y no recibe las confirmaciones (ACKs) de la capa MAC, desencadena un fallo de enlace local. La capa de red detecta la pérdida de conectividad con ese vecino inmediato y redirige el tráfico hacia la siguiente ruta disponible en su tabla de enrutamiento en cuestión de segundos.

### 5.3 ¿Por qué se requiere un enlace A–C de baja calidad (LQ 1) en lugar de un bloqueo total?

Para probar experimentalmente la selección de rutas y la autorrecuperación, es imprescindible que exista una ruta física alternativa. Si el enlace A–C estuviese completamente bloqueado (0 % de conectividad), al apagar el Nodo B no existiría medio físico de propagación entre A y C, provocando una caída total del enlace. En el terreno, los nodos deben desplegarse asegurando un solapamiento de cobertura entre múltiples vecinos para garantizar redundancia real.

### 5.4 ¿Por qué las aplicaciones de SoilSense deben utilizar direcciones ML-EID?

La dirección RLOC cambia dinámicamente cada vez que un nodo se reasocia a la red o cambia su posición lógica en la malla. Si la aplicación de sensado enviara datos a una dirección RLOC, perdería la comunicación tan pronto como la red reestructurara sus rutas. La dirección ML-EID asigna un identificador IPv6 único e invariable al dispositivo basado en su clave de red, garantizando que el tráfico alcance al sensor correcto sin importar qué ruta o router intermedio se utilice.

---

## 6. Mapeo arquitectónico ISO/IEC 30141

Dentro de la arquitectura de referencia para IoT (ISO/IEC 30141), la red malla Thread opera en el **Sensing and Controlling Domain (SCD)**, gestionando los enlaces de radio, el enrutamiento multisalto y la topología física. Los servicios del **Application and Service Domain (ASD)** se comunican directamente con los nodos sensores utilizando sus direcciones ML-EID.

| Rol Thread | Función en la arquitectura | Perfil de consumo energético medido |
|---|---|---|
| **Leader** | Coordina parámetros de red y asignación de Router IDs | Alto (Receptor encendido continuamente / Alimentación fija) |
| **Router** | Reenvía tramas de otros nodos y mantiene tablas de vecinos | Alto (Receptor en escucha activa / Requiere alimentación fija o solar) |
| **REED** | Router elegible en espera de activación por densidad | Moderado-Alto (Pasa a Router activo según la necesidad) |
| **End Device** | Nodo hoja comunicado únicamente con su nodo padre | Moderado (No reenvía tráfico de terceros) |
| **Sleepy End Device (SED)** | Sensor final que apaga su radio y despierta por intervalos | Muy Bajo (Adecuado para sensores con batería de larga duración) |

**Recomendación de diseño:** Para cumplir el objetivo de autonomía de las baterías en el proyecto SoilSense, los nodos configurados como Routers deben contar con alimentación continua (paneles solares o red eléctrica), mientras que los nodos de medición de humedad en suelo deben operar como **Sleepy End Devices (SED)** asociados a un Router cercano.

---

## 7. ADR-002: Selección de topología de red para el campo

**Decisión:** Adoptar **Thread Mesh** como la topología de red principal para el despliegue de sensores de SoilSense en terreno, sujeto a garantizar densidad de routers con alimentación solar.

| Criterio de evaluación | Thread Mesh | Topología Estrella (tipo LoRaWAN) |
|---|---|---|
| **Extensión de Cobertura** | Alta: Se extiende mediante saltos intermedios entre routers | Limitada: Cada nodo depende de la cobertura directa al Gateway |
| **Tolerancia a fallos** | Alta: Se autorrecupera redirigiendo el tráfico (8,7 s medidos) | Baja: La caída del Gateway desconecta a todos sus nodos asociados |
| **Latencia** | Incrementa ~17,4 ms por cada salto adicional | Baja y constante (un solo salto directo) |
| **Consumo en nodos sensores** | Muy bajo en nodos finales (SED); alto en routers | Muy bajo en todos los nodos finales |
| **Complejidad de red** | Gestión automática de rutas mediante MLE y 6LoWPAN | Requerimiento de gestión centralizada en red/gateway |

**Justificación basada en mediciones:** Los datos de laboratorio confirmaron un RTT promedio de 17,2 ms a un salto y 34,6 ms a dos saltos, con un tiempo de autorrecuperación de 8,7 segundos ante la caída de un router. Estos parámetros validan que Thread Mesh proporciona la redundancia y latencia requeridas para la red de monitoreo agrícola.

---

## 8. Resumen operativo para Edwin (Técnico de campo)

- **Tiempo de recuperación medido:** 8,7 segundos promedio (máximo registrado de 11,0 segundos).
- **Cumplimiento de especificación:** Cumple holgadamente el requisito de autorrecuperación inferior a 2 minutos (120 s).
- **Procedimiento de verificación en campo:**
  1. Verificar que los nodos compartan exactamente el mismo *Active Dataset* y operen en el canal 15.
  2. Ejecutar `ot router table` para comprobar que la calidad de enlace entre routers vecinos sea LQ 2 o LQ 3.
  3. Confirmar la existencia de al menos dos routers dentro del alcance de cada zona de sensado para asegurar la ruta de respaldo.
  4. Tras reinsertar o reemplazar un router en campo, verificar su estado mediante `ot state` hasta que confirme el rol de `router`.

---

## 9. Resumen de arquitectura para Samuel Cifuentes

- **Métrica de enrutamiento:** Un enlace de alta calidad (LQ 3) tiene costo 1. La ruta de dos saltos A–B–C presenta un costo de 1 + 1 = 2, siendo seleccionada automáticamente sobre el enlace directo A–C degradado a LQ 1 (Costo = 4).
- **Direccionamiento de aplicación:** Configurar todas las consultas de telemetría y comandos hacia la dirección **ML-EID** del nodo sensor.
- **Justificación:** La dirección ML-EID mantiene la identidad del dispositivo de forma invariable. La dirección RLOC es volátil y cambia cada vez que el nodo se reasocia o conmuta su enrutamiento en la malla.

---

## 10. Conclusiones

1. La pila OpenThread sobre ESP32-C6 selecciona dinámicamente las rutas óptimas basándose en la calidad acumulada de los enlaces (LQI/Costo), prefiriendo rutas de múltiples saltos de alta calidad sobre enlaces directos degradados.
2. El uso de la dirección **ML-EID** es obligatorio para la capa de aplicación de SoilSense, garantizando que el tráfico alcance al sensor objetivo independientemente de las reestructuraciones de la topología.
3. Cada salto de enrutamiento intermedio añade aproximadamente **17,4 ms** de latencia RTT al enlace de radio a 2,4 GHz, un valor perfectamente tolerable para la frecuencia de muestreo de sensores agrícolas.
4. La red demostró capacidad de autorrecuperación en un tiempo promedio de **8,7 segundos** ante la pérdida de un router intermedio, reanudando la transmisión por la ruta de respaldo sin intervención manual.
5. Para optimizar el balance entre cobertura y vida útil de la batería, los nodos Routers deben contar con infraestructura de alimentación continua, permitiendo que los sensores de suelo funcionen como dispositivos finales durmientes (SED).

---

## 11. Referencias

1. Guía de laboratorio, *Lab 2: Thread Mesh — Addressing, Multi-Hop and Self-Healing*, GreenField Technologies — SoilSense.
2. OpenThread CLI Reference: https://openthread.io/reference/cli/commands
3. OpenThread IPv6 Addressing Guide: https://openthread.io/guides/thread-primer/ipv6-addressing
4. OpenThread Router Selection Primer: https://openthread.io/guides/thread-primer/router-selection
5. RFC 6282: *Compression Format for IPv6 Datagrams over IEEE 802.15.4-Based Networks*.
6. ISO/IEC 30141: *Internet of Things (IoT) — Reference Architecture*.
