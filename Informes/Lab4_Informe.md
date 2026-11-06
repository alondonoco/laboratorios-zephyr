# Lab 4 — Sensores Reales: ¿Puedes confiar en la lectura?

**Proyecto:** GreenField Technologies — SoilSense  
**Fase:** Network design  
**Asignatura / laboratorio:** Zephyr + ESP32-C6 + IEEE 802.15.4 / OpenThread  
**Equipo de referencia:** ESP32-C6-DevKitC-1 + Sonda de temperatura DS18B20 (1-Wire)  
**Fecha:** 6 de noviembre de 2026  

---

## 1. Descripción del dispositivo y del objetivo

En este laboratorio se sustituye el valor simulado de temperatura expuesto en el recurso CoAP `/env/temp` por lecturas físicas reales provenientes de una sonda de temperatura DS18B20. El objetivo principal es caracterizar la fiabilidad, precisión y tiempos de respuesta del hardware en un entorno real, así como evaluar la resiliencia del nodo ante fallos físicos (desconexiones accidentales del sensor o pérdida total de energía del nodo). 

Se busca establecer un contrato de datos confiable (Trustworthiness según ISO/IEC 30141) que garantice que los valores reportados al dashboard son precisos y recientes, evitando que el sistema reporte información obsoleta o errónea cuando un componente falla.

---

## 2. Resumen para los interesados

### 2.1 Resumen para Daniela (Agricultora)
**¿El número es correcto?** Sí. Hemos comprobado que el sensor tiene una alta precisión. Es capaz de detectar cambios de hasta 0.0625 °C. Al compararlo con un termómetro de referencia calibrado, la diferencia (offset) fue de tan solo +0.20 °C, lo cual está muy por dentro del margen de error de fábrica. Además, implementamos filtros de seguridad: si el sensor lee temperaturas imposibles para el suelo (como 85 °C o -10 °C), el sistema las rechazará automáticamente.

### 2.2 Resumen para Edwin (Operaciones)
**¿Es el sensor o es el nodo?** Ahora podrás distinguirlo claramente en el dashboard. 
- Si **el sensor se desconecta** de la placa (como ocurrió el martes), el firmware detectará la caída del bus 1-Wire y en un máximo de **10 segundos** enviará un mensaje de error explícito (`{"error": "sensor_fault"}`), dejando de reportar la "última temperatura conocida". 
- Si **el nodo completo pierde energía**, el servidor de recolección dejará de recibir respuestas (timeout de red CoAP). El sistema detectará la ausencia del nodo tras agotar los reintentos de comunicación en aproximadamente **45 segundos**. 

### 2.3 Resumen para Samuel (Arquitecto)
**¿Qué promete ahora `/env/temp`?** El contrato del recurso CoAP ha ganado frescura y confiabilidad. Ahora garantiza lecturas reales cada 10 segundos con un ruido estático inferior a 0.15 °C. En caso de fallo de hardware local, `/env/temp` abandona la retención de datos en caché y retorna un código `5.00 Internal Server Error`, obligando al cliente a registrar un estado de avería en la base de datos en lugar de interpolar valores antiguos.

### 2.4 Resumen para Edward (Seguridad)
**¿Puede alguien inyectar lecturas falsas?** Las validaciones de plausibilidad implementadas hoy (límites de temperatura) descartan lecturas erráticas por fallos eléctricos, pero **no protegen contra manipulación maliciosa**. Si un atacante interviene el cable 1-Wire y emula lecturas válidas (ej. 25 °C), el nodo las aceptará ciegamente. La verificación criptográfica del origen de los datos se abordará en el Lab 6.

---

## 3. Tabla de datos experimentales: Caracterización y tiempos

Se completó la evaluación del sensor físico frente a los parámetros esperados antes de la prueba. 

| Parámetro a medir | Predicción teórica | Valor medido en laboratorio | Objetivo / Referencia |
|---|---|---|---|
| Resolución: el escalón más pequeño | 0.0625 °C | **0.0625 °C** | — (Limitado a 12-bit ADC) |
| Ruido: variación en 1 min (sin tocar) | 0.10 °C | **0.12 °C** | — |
| Offset contra termómetro de referencia | 0.00 °C | **+0.20 °C** | Dentro de la precisión del datasheet (±0.5 °C) |
| Prueba de aliento: tiempo hasta la mitad de subida | 3.0 s | **4.8 s** | — (Afectado por masa térmica de la sonda) |
| Tiempo hasta que el cliente ve el **sensor perdido** | 10 s | **10.2 s** | ≤ 60 s |
| Tiempo hasta que el cliente ve el **nodo perdido** | 45 s | **46.5 s** | > tiempo de sensor (diferenciable) |

> **Nota sobre el offset:** El termómetro de referencia marcó 24.10 °C en la sala, mientras que el DS18B20 reportó un promedio de 24.30 °C. La diferencia es mínima y no requiere calibración por software para la aplicación agrícola.

---

## 4. Auditoría de Confiabilidad (ISO/IEC 30141 §6.6 Trustworthiness)

La confiabilidad (Trustworthiness) del sistema se auditó evaluando la resiliencia y la precisión de los datos (Data Trustworthiness). Según el estándar, un sistema confiable debe manejar los fallos de manera predecible.

- **Detección de integridad de hardware:** El protocolo 1-Wire del DS18B20 retorna un valor de `-127.0 °C` o falla la verificación de paridad (CRC) si hay un problema eléctrico. El firmware ahora mapea este evento a un estado de fallo (Latent to Active fault handling), previniendo la propagación de datos corruptos hacia el *Data Domain*.
- **Plausibilidad de datos:** Se integró un umbral lógico en Zephyr. Un valor de `85.0 °C` (el estado de reset de fábrica por defecto del DS18B20, que suele filtrarse si se lee antes de tiempo) se rechaza explícitamente en la capa de aplicación CoAP.
- **Frescura de datos:** Se eliminó la variable estática que retenía el valor anterior en el handler de CoAP. Ahora la lectura es atómica y *on-demand*.

---

## 5. Procedimiento experimental ejecutado

### Tarea A — Integración del hardware
1. Se conectó la sonda DS18B20 al ESP32-C6. El pin de datos se enrutó al GPIO 4 con una resistencia pull-up externa de 4.7 kΩ hacia los 3.3V.
2. Se modificó el `device tree` (DTS) de Zephyr en `app.overlay` para habilitar el bus de 1-Wire y registrar el sensor bajo la API de sensores de Zephyr.
3. Se compiló y flasheó el firmware modificado y se verificó mediante el shell de comandos (`sensor get temp`) que la interfaz leía correctamente.

### Tarea B — Caracterización del sensor
1. **Ruido y resolución:** Se dejó la placa estática sobre la mesa durante 1 minuto. Mediante un script, se solicitaron datos cada 2 segundos. Se registraron fluctuaciones mínimas de un bit LSB (0.0625 °C), marcando una oscilación total (spread) de 0.12 °C.
2. **Offset:** Se comparó la temperatura estable del sensor con un higrómetro/termómetro calibrado de laboratorio.
3. **Prueba de aliento (Response time):** Se registró la temperatura base (24.3 °C). Se aplicó calor y humedad exhalando directamente sobre la cápsula de acero del sensor. La temperatura pico alcanzó 33.5 °C (Delta = 9.2 °C). El punto medio de subida (28.9 °C) se alcanzó a los 4.8 segundos. La latencia se debe a la cápsula metálica de protección, ideal para enterrar en suelo, pero más lenta que un sensor al aire desnudo.

### Tarea C — Simulación de fallos operacionales
Se configuró un cliente CoAP externo sondeando `/env/temp` cada 10 segundos.

1. **Fallo de sensor:** Se desconectó físicamente el pin de datos (GPIO 4) en pleno funcionamiento. En el siguiente ciclo de sondeo (a los 10.2 s), el cliente recibió un `5.00 Internal Server Error` en lugar de los 24.3 °C que antes se quedaban "congelados" en memoria.
2. **Fallo de nodo (Pérdida de energía):** Se reconectó el sensor y luego se desconectó el cable USB que alimentaba al ESP32-C6. El cliente CoAP intentó realizar su petición GET habitual, no recibió el ACK de la capa de red y, tras agotar los 4 reintentos exponenciales definidos en el estándar CoAP, declaró un *Timeout* a los 46.5 segundos. 
3. **Conclusión:** Ambos fallos tienen "firmas" distintas. Un error de aplicación (5.00) significa hardware de medición dañado pero conectividad viva. Un *Timeout* de red significa caída total del nodo o de la red Thread.

---

## Referencias

1. Maxim Integrated (Dallas). *DS18B20 Programmable Resolution 1-Wire Digital Thermometer Datasheet*.
2. ISO/IEC 30141:2024, *Internet of Things (IoT) — Reference Architecture*, Clause 6.6 (Trustworthiness).
3. Zephyr Project Documentation, *1-Wire bus and DS18B20 sensor API*.
4. RFC 7252, *The Constrained Application Protocol (CoAP)*, Section 4.2 (Message Transmission).
