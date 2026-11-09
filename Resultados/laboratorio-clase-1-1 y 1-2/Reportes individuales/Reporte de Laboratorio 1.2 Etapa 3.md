---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.2 — Resiliencia en `catalog-service` |
| **Etapa** | Paso 2.4: `@Timeout(800)` + `@Retry(maxRetries = 2, delay = 150)` + `@Fallback` + **`@CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.7, delay = 5000)`** |
| **Servicio evaluado** | `catalog-service` (proyecto-base-unidad-01) |
| **Endpoint** | `GET http://localhost:8080/v1/products` |
| **Herramienta** | `test-availability.sh` (curl + bash) |
| **Solicitudes ejecutadas** | 200 |
| **Archivo de evidencia** | `availability-modification3.txt` |
| **Entorno** | Ubuntu 26.04.1 LTS, OpenJDK 25, Maven 3.9.12, Quarkus Dev Mode |

## 2. Objetivo de la etapa

Medir el impacto de añadir `@CircuitBreaker` sobre la configuración acumulada de las etapas 1 y 2. El objetivo es verificar que:

- La disponibilidad se mantiene en **1.0000** (no se sacrifica nada respecto a la Etapa 2).
- El **tiempo medio y máximo de las respuestas degradadas se reduce** notablemente al evitar reintentos innecesarios cuando el backend está claramente fuera de servicio.
- En los logs aparecen las **transiciones de estado** del circuito: rachas con contador `0` (OPEN) alternadas con intentos reales (CLOSED / HALF_OPEN).

## 3. Metodología

1. Se añadió `@CircuitBreaker(requestVolumeThreshold = 10, failureRatio = 0.7, delay = 5000)` sobre `getProducts()`, manteniendo `@Timeout`, `@Retry` y `@Fallback`.
2. Se recompiló y reinició la aplicación en modo dev.
3. Se ejecutó el script reutilizable:

```bash
./test-availability.sh availability-modification3.txt
```

4. Se analizaron las métricas agregadas y se compararon contra las etapas previas.

## 4. Resultados obtenidos

### 4.1 Métricas agregadas

| Métrica | Valor |
|---|---|
| Respuestas `SUCCESS` | **20** |
| Respuestas `DEGRADED_CACHE` | **180** |
| Respuestas HTTP 200 | **200** |
| Respuestas HTTP ≠ 200 (errores 500) | **0** |
| Solicitudes totales | **200** |
| **Disponibilidad observada** | **1.0000 (100.00 %)** |

### 4.2 Salida impresa por el script

```text
Resultados guardados en: availability-modification3.txt

Respuestas SUCCESS: 20
Respuestas HTTP 200: 200
Respuestas HTTP distintas de 200: 0
Solicitudes totales: 200
Disponibilidad observada: 1.0000
```

> **Nota:** el script del Lab 1.1 solo cuenta `SUCCESS` y HTTP 200; no cuenta `DEGRADED_CACHE`. Por eso `SUCCESS = 20` pero `HTTP 200 = 200`. La diferencia (180) son las respuestas degradadas, ahora servidas **sin reintentar** en su mayoría gracias al CircuitBreaker.

### 4.3 Fragmento del archivo `availability-modification3.txt`

Se muestran tres patrones claramente distinguibles: éxito directo, degradación con reintentos completos (~1.0–1.2 s, fase CLOSED) y degradación casi instantánea (~0.02–0.6 s, fase OPEN).

```text
001 200 0.163934 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
002 200 0.008139 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
003 200 1.171040 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}   ← CLOSED: 3 intentos
004 200 1.002655 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}   ← CLOSED: 3 intentos
005 200 0.008389 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
006 200 0.316315 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}   ← transición / HALF_OPEN
007 200 0.483272 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
008 200 0.500786 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
...
019 200 0.024402 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}   ← OPEN: rechazo rápido
020 200 0.611279 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
...
174 200 0.315128 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
175 200 1.233345 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}   ← CLOSED de nuevo
...
200 200 0.351600 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
```

### 4.4 Fragmento de logs del servidor (Terminal 1)

![Fragmento de logs del servidor](Fragmento%20de%20logs%20del%20servidor%20m3.png)

## 5. Análisis de resultados

### 5.1 Disponibilidad intacta

La disponibilidad se mantiene en **1.0000 (100 %)**, igual que en la Etapa 2. Añadir `@CircuitBreaker` **no degrada la disponibilidad**: los 0 errores HTTP se conservan y las 200 solicitudes obtienen HTTP 200.

### 5.2 Caída del `SUCCESS` directo (77 → 20)

El número de respuestas `SUCCESS` bajó de **77** (Etapa 2) a **20** (Etapa 3). Esto es **esperado y pedagógicamente correcto**:

- En la Etapa 2, cada solicitud tenía una probabilidad independiente de ~38 % de resolver en el intento 1.
- En la Etapa 3, el CircuitBreaker **abre el circuito** tras 7 fallos en las últimas 10 solicitudes. Mientras está OPEN (~5 s), **todas** las solicitudes se rechazan sin ejecutar `getProducts()`, por lo que el intento 1 ni siquiera llega a la BD primaria.
- Resultado: durante la mayor parte de la prueba el circuito está OPEN y las solicitudes van directo al fallback, reduciendo drásticamente las oportunidades de `SUCCESS`.

### 5.3 Reducción drástica de latencia en respuestas degradadas

Comparando `time_total` de las solicitudes `DEGRADED_CACHE`:

| Rango de latencia | Etapa 2 (sin CB) | Etapa 3 (con CB) | Interpretación |
|---|---|---|---|
| ~0.80 – 1.45 s | Mayoría de degradadas | Algunas (solicitudes 003, 004, 023, 024, 033, 175, 178, 179, 196–198) | **Fase CLOSED**: los 3 intentos se ejecutan completos |
| ~0.15 – 0.65 s | Prácticamente inexistente | **Mayoría** de degradadas | **Fase OPEN**: rechazo sin reintentos, solo latencia del fallback |
| ~0.02 – 0.15 s | Inexistente | Varias (019, 042, 052, 066, 071, 100, 105, 123, 136, 141…) | **Fase OPEN** con `ThreadLocal` ya limpios |
| Máximo | **1.453568 s** | **1.246826 s** | Reducción del pico |

**Conclusión clave:** la latencia media de las respuestas degradadas cae de ~1.1 s a ~0.35 s. Esto es exactamente el beneficio esperado del CircuitBreaker: **protege al backend de reintentos innecesarios** cuando está claramente caído, y **responde al cliente mucho más rápido**.

### 5.4 Detección de las fases del circuito en el archivo

Aunque las transiciones de estado no se imprimen como tal en el archivo, se pueden inferir:

| Fase | Evidencia en `availability-modification3.txt` |
|---|---|
| **CLOSED inicial** | Solicitudes 003, 004 con latencia ~1.0–1.2 s (3 intentos completos) |
| **Apertura (OPEN)** | A partir de ~solicitud 006, las latencias caen a 0.3–0.6 s y luego a 0.02–0.15 s (rechazos sin intentos) |
| **OPEN sostenido** | Bloque de solicitudes 034–076 con latencias 0.1–0.6 s, sin `SUCCESS` directos |
| **HALF_OPEN / prueba** | Aparición de `SUCCESS` esporádicos (077, 079, 081, 174, 176, 177, 194, 195) que indican que el circuito probó y el backend respondió |
| **Reapertura / CLOSED** | Nuevos bloques con latencia ~1.0–1.2 s (175, 178, 179, 196–198) al volver a fallar |

### 5.5 Comparación consolidada de las tres etapas del Lab 1.2

| Métrica | Lab 1.1 (base) | Etapa 1 (T+R) | Etapa 2 (+FB) | **Etapa 3 (+CB)** |
|---|---|---|---|---|
| `SUCCESS` (HTTP 200) | 131 (65.5 %) | 87 (43.5 %) | 77 (38.5 %) | **20 (10.0 %)** |
| `DEGRADED_CACHE` (HTTP 200) | 0 | 0 | 123 (61.5 %) | **180 (90.0 %)** |
| HTTP 500 | 69 (34.5 %) | 113 (56.5 %) | 0 (0 %) | **0 (0 %)** |
| **Disponibilidad** | 0.6550 | 0.4350 | 1.0000 | **1.0000** |
| Latencia máx. | ~3.01 s | ~1.97 s | ~1.45 s | **~1.25 s** |
| Latencia media degradada | N/A | N/A | ~1.1 s | **~0.35 s** |

## 6. Conclusiones de la etapa

1. **Disponibilidad garantizada y preservada:** 100 % de HTTP 200, sin errores 500, igual que en la Etapa 2.
2. **CircuitBreaker protege al backend:** al abrir el circuito tras 7 fallos en 10 solicitudes, se evitan reintentos innecesarios durante ~5 s, lo que se refleja en la **caída de la latencia media de respuestas degradadas de ~1.1 s a ~0.35 s**.
3. **Menos `SUCCESS` directos es señal de protección activa:** el descenso de 77 a 20 se explica porque, con el circuito OPEN, muchas solicitudes ya no llegan a la BD primaria. **No es una regresión**: es exactamente lo que el CircuitBreaker debe hacer.
4. **Latencia máxima reducida:** de ~1.45 s a ~1.25 s, porque las fases OPEN evitan que se acumulen los tres intentos con sus delays.
5. **Implementación completa de resiliencia:** `Timeout` + `Retry` + `Fallback` + `CircuitBreaker` conviven correctamente y el sistema prioriza **disponibilidad + rapidez** sobre consistencia, con un trade-off explícito y medible.

## 7. Evidencia entregable de la etapa

- `availability-modification3.txt` (200 líneas, formato `NNN <http_code> <time_total> <body>`).
- Salida del script con las 5 métricas agregadas.
- Fragmento de logs del servidor mostrando la transición de estados del CircuitBreaker: rachas de `Fallback activado: Se agotaron todos los 3 intentos` (CLOSED) → rachas de `Fallback activado: Se agotaron todos los 0 intentos` sin líneas `[Intento ...]` previas (OPEN) → nuevos `[Intento 1/3]` cortos (HALF_OPEN) (a completar en la sección 4.4).
