# Lab 5 — Border Router: Conectando la red Mesh a Internet

**Proyecto:** GreenField Technologies — SoilSense  
**Fase:** Integration  
**Asignatura / laboratorio:** Zephyr + ESP32-C6 + ESP32-S3 + IEEE 802.15.4 / OpenThread  
**Equipo de referencia:** ESP32-C6 (RCP y Nodos S/C) + ESP32-S3-DevKitC-1 (Host OTBR)  
**Fecha:** 20 de noviembre de 2026  

---

## 1. Descripción del dispositivo y del objetivo

En este laboratorio se integró un **OpenThread Border Router (OTBR)** a la red Thread existente, estableciendo un puente bidireccional entre la red inalámbrica 802.15.4 (Mesh) y la red Wi-Fi (Internet). El OTBR se construyó mediante una arquitectura de coprocesador, utilizando un ESP32-S3 para el manejo de Wi-Fi y la pila IP, conectado por UART a un ESP32-C6 configurado como *Radio Co-Processor* (RCP).

El objetivo principal es demostrar que los recursos de la capa de aplicación CoAP (`/env/temp` y `/act/valve`) pueden ser consultados y manipulados desde un dispositivo externo a la red Mesh (como un teléfono móvil o un servidor local), evaluando al mismo tiempo la latencia que introduce el enrutamiento de borde y verificando la resiliencia operativa del sistema cuando el enlace externo falla.

---

## 2. Resumen para los interesados

### 2.1 Bitácora de laboratorio: Mensaje para Daniela (Agricultora)
**¿Puede mi teléfono ver el campo?** Sí. Ahora, conectada al mismo punto de acceso Wi-Fi que el enrutador de borde, puedes ver la temperatura del suelo y abrir o cerrar la válvula directamente desde el navegador de tu teléfono. La respuesta es casi instantánea (el retraso es de menos de 50 milisegundos, imperceptible para este tipo de acciones).

**¿Qué pasa si el enrutador (OTBR) se apaga o se daña?** Hicimos la prueba de desconectarlo abruptamente (Tarea F). Como era de esperarse, perdiste temporalmente el acceso desde el teléfono. Sin embargo, **el campo sigue funcionando de manera autónoma**. Probamos enviar el comando a la válvula desde otro nodo sensor ubicado físicamente en el cultivo, y la válvula respondió de inmediato. La red de sensores no depende del Wi-Fi ni de Internet para sus rutinas locales.

---

## 3. ADR-005 — Selección de un único OTBR para el piloto

**Estado:** Decisión adoptada.  
**Contexto:** Se requiere conectar la red Thread al sistema de control centralizado vía Wi-Fi. Evaluar si un solo OTBR es suficiente para la fase piloto o si se requiere redundancia inmediata.

### 3.1 Datos experimentales de latencia como justificación
Durante las pruebas, el enrutamiento de paquetes entre Wi-Fi y Thread a través del OTBR introdujo una latencia añadida de apenas **26 ms**. La solicitud CoAP completa de extremo a extremo tardó un promedio de **48 ms**, muy por debajo del objetivo máximo de 200 ms. 

### 3.2 Decisión
**ADR-005: Desplegar un único OTBR para la fase piloto.** 
*Justificación:* El rendimiento térmico y de latencia del ESP32-S3 como OTBR es excelente para el volumen de tráfico actual. Aceptamos este componente como un **punto único de fallo exclusivo para la telemetría externa (off-mesh reach)**. Dado que las interacciones críticas de control dentro de la parcela (ej. comunicación de sensor a válvula) retienen su operabilidad local mediante direcciones `mesh-local`, la falta de redundancia en la salida a Internet no compromete la seguridad física del cultivo durante el piloto.

---

## 4. Mapeo arquitectónico ISO/IEC 30141

De acuerdo con el Anexo A de la norma ISO/IEC 30141, el OTBR se clasifica en el *Enterprise system pattern* (Figura A.5) como un **SCD-hosted IoT gateway** (pasarela alojada en el dominio de Sensado y Control). El OTBR no asume funciones de servidor de aplicaciones; su rol es estrictamente de red.

### 4.1 Evaluación de funciones de pasarela (Tabla A.3 Scorecard)
| Función de IoT Gateway | ¿Lo hace nuestro OTBR actual? | Evidencia / Justificación |
|---|---|---|
| **Conversión de protocolo** | **Sí** | Transfiere paquetes IP entre tramas 802.11 (Wi-Fi) y 802.15.4 (Thread). |
| **Mapeo de direcciones** | **Sí** | NAT64 activo, traduciendo IPs IPv6 de Thread a IPv4 hacia Internet. |
| Procesamiento de datos | No | Los payloads CoAP pasan inalterados (routing puro en capa de red). |
| Fusión de información | No | No agrega ni promedia lecturas de múltiples sensores. |
| Certificación / Seguridad | No | Actualmente carece de autenticación de clientes externos (ver §7). |
| Gestión de equipos | Parcial | Anuncia el prefijo OMR, pero no gestiona los ciclos de vida de los nodos. |

### 4.2 Cadenas de Red (Tabla A.4 - Enterprise networking pattern)
| Red (Tabla A.4) | En SoilSense | Direcciones observadas (Ejemplo empírico) |
|---|---|---|
| **Proximity** | La malla Thread (Nodos S, C y RCP) | `fd5a:8c9f:1122:0:4321:...` (mesh-local), `fde7:b192:...` (OMR) |
| **Access** | El Wi-Fi del OTBR y el Hotspot | `192.168.137.105` (IPv4) / `2800:bf0:8000:...` (IPv6 global Wi-Fi) |
| **Services** | Laptop con el dashboard CoAP | `192.168.137.42` |
| **User** | Teléfono móvil de Daniela | IP privada del celular en la red del Hotspot |

---

## 5. Rendimiento: Tabla de mediciones

| Parámetro a medir | Predicción | Valor medido | Objetivo / Resultado |
|---|---:|---:|---|
| RTT en la malla, C → S (`ot ping`) | 20 ms | **22 ms** | Consistente con Lab 3 |
| Salto Wi-Fi, Laptop → OTBR (Power save ON) | 50 ms | **47 ms** | Alta varianza por modo reposo |
| Salto Wi-Fi, Laptop → OTBR (Power save OFF) | 10 ms | **8 ms** | — |
| Laptop → Nodo S (Mesh), `ping` | 35 ms | **34 ms** | — |
| Laptop → Nodo S, CoAP GET `/env/temp` | 60 ms | **48 ms** | **< 200 ms** (Cumple) |
| **Lab 5: Latencia añadida por OTBR** | < 100 ms | **26 ms** | **< 100 ms** (Cumple) |
| **Lab 5: RTT NAT64 (S → 8.8.8.8)** | 120 ms | **114 ms** | **< 200 ms** (Cumple) |
| OTBR APAGADO: C → S `PUT /act/valve` | Funciona | **Funciona** | El tráfico interno no se afecta |

*Nota de cálculo de latencia:* La latencia añadida por el OTBR (26 ms) se obtuvo restando el ping Laptop→OTBR (8 ms) del ping Laptop→S (34 ms).

---

## 6. Análisis técnico: Principios Fundamentales

**1. ¿Por qué el anuncio de un prefijo más SLAAC hace que "S" sea alcanzable desde el exterior sin "magia" de pasarelas especiales?**  
Porque OpenThread se basa en IPv6 nativo de extremo a extremo. El OTBR simplemente difunde un prefijo OMR (Off-Mesh Routable) a la red. El nodo S usa SLAAC (Stateless Address Autoconfiguration) para combinar ese prefijo con su propia dirección MAC, formando una dirección IP globalmente única. Desde ese momento, el OTBR solo realiza enrutamiento IP estándar (Capa 3); no requiere proxies a nivel de aplicación ni mapeos de puertos complejos.

**2. ¿Por qué se necesita NAT64 si Thread es IPv6?**  
Aunque Thread y nuestros nodos operan puramente en IPv6, gran parte de la infraestructura externa (como puntos de acceso Wi-Fi antiguos, proveedores de Internet de telefonía o servidores IPv4 heredados como `8.8.8.8`) solo soportan IPv4. NAT64 actúa como un traductor en el OTBR: envuelve y traduce los paquetes IPv6 originados en la red Thread a paquetes IPv4 para que puedan transitar por la red Access e Internet.

**3. ¿Por qué los bytes de `/env/temp` son los mismos a ambos lados del OTBR?**  
Porque el OTBR opera en las capas 2 y 3 (MAC y Red). Su única función es desencapsular el paquete IP de la trama 802.11 (Wi-Fi) y reencapsularlo en la trama 802.15.4 (Thread), y viceversa. La capa de aplicación (CoAP, donde reside `/env/temp`) no es alterada. Esto confirma la naturaleza transparente del protocolo.

**4. ¿Por qué una pasarela conectada a la red eléctrica debe desactivar el "Wi-Fi Power Save", cuando un nodo de batería haría lo contrario?**  
El ahorro de energía (Power Save) en Wi-Fi funciona apagando el receptor de radio temporalmente e indicándole al Access Point que guarde en un buffer los paquetes dirigidos a él. Esto introduce fluctuaciones de latencia significativas (observadas en 47 ms, pero pueden superar los 100 ms). Dado que el OTBR es el embudo de todo el tráfico bidireccional y está alimentado por la red eléctrica, debe mantener su radio permanentemente activo (`wifi ps off`) para despachar los paquetes hacia los nodos Mesh instantáneamente, reduciendo la latencia (bajó a 8 ms en nuestras mediciones).

---

## 7. Ética, Seguridad y Sostenibilidad

- **Sostenibilidad operacional (Resiliencia):** La Tarea F proporcionó evidencia clave. Al apagar intencionalmente el OTBR, el enrutamiento hacia el laptop (exterior) colapsó, pero el enrutamiento interno en la malla Thread (`mesh-local`) se mantuvo intacto. Un operario de campo (o automatismo) dentro de la malla conserva la capacidad de accionar la válvula, previniendo desperdicios hídricos catastróficos si falla la conexión a Internet.
- **Seguridad física:** Actualmente, cualquier persona que conecte un cable USB al ESP32-S3 (OTBR) obtiene acceso root a la terminal shell de OpenThread sin ningún tipo de autenticación. Este es un vector de ataque severo si el OTBR se instala en un poste de acceso público.
- **Privacidad y acceso a la información:** Al unir ambas redes, hemos expuesto los recursos CoAP (`/env/temp` y `/act/valve`) a la red Wi-Fi. Como no se ha implementado cifrado a nivel de aplicación (CoAP puro en lugar de OSCORE o DTLS), **cualquier usuario conectado al mismo hotspot de Daniela puede leer la temperatura y accionar la válvula sin credenciales**. Esta vulnerabilidad de acceso es el desafío inmediato a resolver en el próximo ciclo (Lab 6).
