---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.2 — Resiliencia en `catalog-service` |
| **Etapa** | Paso 2.2: `@Timeout(800)` + `@Retry(maxRetries = 2, delay = 150)` |
| **Servicio evaluado** | `catalog-service` (proyecto-base-unidad-01) |
| **Endpoint** | `GET http://localhost:8080/v1/products` |
| **Herramienta** | `test-availability.sh` (curl + bash) |
| **Solicitudes ejecutadas** | 200 |
| **Archivo de evidencia** | `availability-modification1.txt` |
| **Entorno** | Ubuntu 26.04.1 LTS, OpenJDK 25, Maven 3.9.12, Quarkus Dev Mode |

## 2. Objetivo de la etapa

Medir el impacto de los dos primeros patrones de resiliencia sobre el `catalog-service` antes de introducir `@Fallback` y `@CircuitBreaker`:

- **`@Timeout(800)`**: cancela cualquier invocación que exceda 800 ms, evitando bloqueos de ~3 s.
- **`@Retry(maxRetries = 2, delay = 150)`**: hasta 3 intentos por solicitud (1 original + 2 reintentos) con 150 ms de espera fija entre intentos.

Esta etapa **no incluye respuesta degradada**: si los 3 intentos fallan, el cliente recibe HTTP 500. El objetivo es aislar el efecto de `Timeout` + `Retry` antes de añadir `Fallback`.

## 3. Metodología

1. Se añadieron los campos `ThreadLocal` (`ATTEMPT_COUNT`, `SHOULD_FAIL_FIRST_ATTEMPTS`) para rastrear reintentos por solicitud.
2. Se anotó `getProducts()` con `@Timeout(800)` y `@Retry(maxRetries = 2, delay = 150)`.
3. Se recompiló y reinició la aplicación en modo dev.
4. Se ejecutó el script reutilizable del Lab 1.1:

```bash
./test-availability.sh availability-modification1.txt
```

5. Se analizaron las métricas agregadas por el propio script.

## 4. Resultados obtenidos

### 4.1 Métricas agregadas

| Métrica | Valor |
|---|---|
| Respuestas `SUCCESS` | **87** |
| Respuestas HTTP 200 | **87** |
| Respuestas HTTP ≠ 200 (errores 500) | **113** |
| Solicitudes totales | **200** |
| **Disponibilidad observada** | **0.4350 (43.50 %)** |

### 4.2 Salida impresa por el script

```text
Resultados guardados en: availability-modification1.txt

Respuestas SUCCESS: 87
Respuestas HTTP 200: 87
Respuestas HTTP distintas de 200: 113
Solicitudes totales: 200
Disponibilidad observada: .4350
```

### 4.3 Fragmento del archivo `availability-modification1.txt`

Se muestran tres solicitudes representativas: dos fallos con `RuntimeException` tras reintentos y una con `TimeoutException` capturada por `@Timeout`. Las trazas de stack se abreviaron con `[...]`.

```text
001 500 0.983884  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout after retries
002 500 1.083939  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout after retries
003 500 1.097364  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout after retries
004 200 0.017885 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
...
128 500 1.452046  500 - Internal Server Error [...] org.eclipse.microprofile.faulttolerance.exceptions.TimeoutException: com.ecom.catalog.CatalogResource#getProducts timed out
...
200 500 0.964101  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout after retries
```

### 4.4 Fragmento de logs del servidor (Terminal 1)

![Fragmento de logs del servidor](Fragmento%20de%20logs%20del%20servidor%20m1.png)

## 5. Análisis de resultados

### 5.1 Efecto de `@Timeout(800)` sobre la latencia

En el Lab 1.1 se observaron solicitudes con `time_total` cercano a **3.0 s**. En esta etapa, **el valor máximo ya no supera ~1.9 s** (la solicitud 177 con `1.965517` es la más alta del archivo). Las solicitudes que caen en la rama de latencia simulada (3000 ms) ahora son **canceladas por `@Timeout` a los 800 ms** y registradas como `TimeoutException`, como se ve en las solicitudes 128, 129, 177–181.

Esto confirma el ahorro esperado de ~2200 ms por solicitud afectada y elimina el bloqueo indefinido del hilo.

### 5.2 Efecto de `@Retry` sobre el patrón de fallos

Con la nueva lógica, cada solicitud decide en el intento 1 (60 % de probabilidad) si **fallará en los 3 intentos**. En esos casos:

- **Intento 1** → `RuntimeException: Database connection timeout`
- **Intento 2** → `Thread.sleep(3000)` → **capturado por `@Timeout(800)`** → `TimeoutException`
- **Intento 3** → `RuntimeException: Database connection timeout after retries`

Por eso el archivo mezcla dos tipos de error 500: `RuntimeException after retries` (mayoría) y `TimeoutException` (solicitudes donde el intento 2 fue el último registrado antes de agotar reintentos). Ambas son evidencia de que `@Retry` se ejecuta y de que `@Timeout` corta la latencia simulada.

### 5.3 Comparación con la línea base (Lab 1.1)

| Métrica | Lab 1.1 (sin patrones) | Lab 1.2 – Etapa 1 (Timeout + Retry) | Cambio |
|---|---|---|---|
| HTTP 200 `SUCCESS` | 131 (65.50 %) | **87 (43.50 %)** | ↓ 22 pp |
| HTTP 500 | 69 (34.50 %) | **113 (56.50 %)** | ↑ 22 pp |
| `time_total` máximo | ~3.01 s | **~1.97 s** | ↓ ~35 % |
| Latencia pico por solicitud | 3000 ms bloqueados | 800 ms + reintentos | ↓ ~2200 ms |

**Interpretación:** la disponibilidad **bajó** respecto a la línea base porque el modelo de fallos de esta etapa es más agresivo (60 % de solicitudes fallan en los 3 intentos, frente al 30 % de fallos independientes del Lab 1.1). No es una regresión del servicio, sino un **cambio de escenario pedagógico** que prepara el terreno para que `@Fallback` demuestre su impacto en la siguiente etapa.

### 5.4 Ausencia de `DEGRADED_CACHE`

No aparece ningún estado `DEGRADED_CACHE` en `availability-modification1.txt`, lo cual es **correcto y esperado**: esta etapa aún no incluye `@Fallback`. Los 113 errores HTTP 500 son exactamente las solicitudes que, en la etapa 2, serán "salvadas" por `getCatalogFallback()` y transformadas en HTTP 200 con datos en caché.

## 6. Conclusiones de la etapa

1. **`@Timeout(800)` funciona:** la latencia máxima observada cae de ~3.0 s a ~1.97 s, eliminando bloqueos largos por congestión de BD.
2. **`@Retry` funciona:** los logs y el archivo muestran los tres intentos por solicitud con los tipos de fallo alternados (`RuntimeException` → `TimeoutException` → `RuntimeException after retries`).
3. **`Timeout` + `Retry` solos no restauran la disponibilidad:** 113 de 200 solicitudes (56.5 %) siguen retornando HTTP 500 porque no existe respuesta degradada.
4. **Punto de partida para la etapa 2:** los 113 errores HTTP 500 son el objetivo directo de `@Fallback`. La expectativa es que, al añadir `getCatalogFallback()`, esos errores se conviertan en HTTP 200 `DEGRADED_CACHE` y la disponibilidad salte a ~1.0000.

## 7. Evidencia entregable de la etapa

- `availability-modification1.txt` (200 líneas, formato `NNN <http_code> <time_total> <body>`).
- Salida del script con las 5 métricas agregadas.
- Fragmento de logs del servidor mostrando la secuencia `[Intento 1/3] … [Intento 2/3] … [Intento 3/3] …` para solicitudes fallidas, y `[Intento 1/3] Catálogo retornado exitosamente…` para las exitosas (a completar en la sección 4.4).