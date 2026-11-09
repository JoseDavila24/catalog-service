---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.2 — Resiliencia en `catalog-service` |
| **Etapa** | Paso 2.3: `@Timeout(800)` + `@Retry(maxRetries = 2, delay = 150)` + `@Fallback(fallbackMethod = "getCatalogFallback")` |
| **Servicio evaluado** | `catalog-service` (proyecto-base-unidad-01) |
| **Endpoint** | `GET http://localhost:8080/v1/products` |
| **Herramienta** | `test-availability.sh` (curl + bash) |
| **Solicitudes ejecutadas** | 200 |
| **Archivo de evidencia** | `availability-modification2.txt` |
| **Entorno** | Ubuntu 26.04.1 LTS, OpenJDK 25, Maven 3.9.12, Quarkus Dev Mode |

## 2. Objetivo de la etapa

Medir el impacto de añadir `@Fallback(fallbackMethod = "getCatalogFallback")` sobre la configuración de la etapa 1. El objetivo es verificar que:

- Los errores HTTP 500 (que en la etapa 1 llegaban al cliente) se transformen en HTTP 200 con `DEGRADED_CACHE`.
- La disponibilidad observada alcance ~1.0000.
- El cliente **nunca** reciba un error HTTP, incluso cuando el backend falla en los 3 intentos.

## 3. Metodología

1. Se añadió el método `getCatalogFallback()` y la anotación `@Fallback(fallbackMethod = "getCatalogFallback")` sobre `getProducts()`.
2. Se ajustó el `finally` de `getProducts()` para **no resetear** los `ThreadLocal` antes de que el fallback los lea (así el contador de intentos agotados queda visible en los logs).
3. Se recompiló y reinició la aplicación en modo dev.
4. Se ejecutó el script reutilizable:

```bash
./test-availability.sh availability-modification2.txt
```

5. Se analizaron las métricas agregadas por el propio script y se comparó contra `availability-baseline.txt` (Lab 1.1) y `availability-modification1.txt` (Etapa 1).

## 4. Resultados obtenidos

### 4.1 Métricas agregadas

| Métrica | Valor |
|---|---|
| Respuestas `SUCCESS` | **77** |
| Respuestas `DEGRADED_CACHE` | **123** |
| Respuestas HTTP 200 | **200** |
| Respuestas HTTP ≠ 200 (errores 500) | **0** |
| Solicitudes totales | **200** |
| **Disponibilidad observada** | **1.0000 (100.00 %)** |

### 4.2 Salida impresa por el script

```text
Resultados guardados en: availability-modification2.txt

Respuestas SUCCESS: 77
Respuestas HTTP 200: 200
Respuestas HTTP distintas de 200: 0
Solicitudes totales: 200
Disponibilidad observada: 1.0000
```

> **Nota:** el script del Lab 1.1 solo cuenta `SUCCESS` y HTTP 200; no cuenta `DEGRADED_CACHE`. Por eso `SUCCESS = 77` pero `HTTP 200 = 200`. La diferencia (123) son exactamente las respuestas degradadas servidas por el fallback.

### 4.3 Fragmento del archivo `availability-modification2.txt`

Se muestran solicitudes representativas de los dos escenarios ahora posibles: éxito directo desde BD primaria y respuesta degradada desde caché.

```text
001 200 0.144662 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
002 200 0.005937 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
003 200 1.009062 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
004 200 1.182183 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
005 200 0.007111 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
006 200 0.006546 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
007 200 1.202335 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
...
198 200 0.888305 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
199 200 1.160081 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
200 200 1.214051 {"status":"DEGRADED_CACHE","data":["Cached Product A","Cached Product B"]}
```

### 4.4 Fragmento de logs del servidor (Terminal 1)

![Fragmento de logs del servidor](Fragmento%20de%20logs%20del%20servidor%20m2.png)

## 5. Análisis de resultados

### 5.1 Disponibilidad garantizada

La disponibilidad saltó de **43.50 %** (Etapa 1) a **100.00 %** (Etapa 2). Los **113 errores HTTP 500** de la Etapa 1 desaparecieron por completo: **0 respuestas HTTP ≠ 200** en las 200 solicitudes. Cada solicitud que agota los 3 intentos es interceptada por `@Fallback` y devuelve HTTP 200 con `DEGRADED_CACHE`.

### 5.2 Distribución `SUCCESS` vs `DEGRADED_CACHE`

- **77 SUCCESS (38.5 %):** solicitudes que resolvieron en el intento 1 contra la BD primaria.
- **123 DEGRADED_CACHE (61.5 %):** solicitudes que agotaron reintentos y fueron servidas desde caché.

La proporción `SUCCESS ≈ 38.5 %` es coherente con el modelo de fallos de la etapa (60 % de solicitudes marcadas para fallar en los 3 intentos). Las 123 degradadas son exactamente las solicitudes que en la Etapa 1 terminaban en HTTP 500.

### 5.3 Latencia

- **Ruta `SUCCESS`:** `time_total` entre **2.8 ms y ~145 ms** (típicamente < 10 ms). Sin cambios respecto a la Etapa 1.
- **Ruta `DEGRADED_CACHE`:** `time_total` entre **~0.80 s y ~1.45 s**. Este rango corresponde a la suma de: intento 1 (~instantáneo con `RuntimeException`) + `delay=150 ms` + intento 2 cortado por `@Timeout(800)` + `delay=150 ms` + intento 3 (~instantáneo) + invocación del fallback. **Ya no hay ninguna solicitud cercana a 3.0 s.**
- **Latencia máxima observada:** `1.453568 s` (solicitud 046), muy por debajo de los ~3.0 s del Lab 1.1.

### 5.4 Comparación consolidada

| Métrica | Lab 1.1 (línea base) | Etapa 1 (Timeout + Retry) | **Etapa 2 (+ Fallback)** |
|---|---|---|---|
| `SUCCESS` (HTTP 200) | 131 (65.5 %) | 87 (43.5 %) | **77 (38.5 %)** |
| `DEGRADED_CACHE` (HTTP 200) | 0 | 0 | **123 (61.5 %)** |
| HTTP 500 | 69 (34.5 %) | 113 (56.5 %) | **0 (0 %)** |
| **Disponibilidad** | 0.6550 | 0.4350 | **1.0000** |
| Latencia máx. | ~3.01 s | ~1.97 s | **~1.45 s** |

## 6. Conclusiones de la etapa

1. **`@Fallback` elimina los HTTP 500:** los 113 errores de la Etapa 1 se transformaron íntegramente en respuestas HTTP 200 con `DEGRADED_CACHE`.
2. **Disponibilidad al 100 %:** la métrica pasó de 0.4350 a **1.0000**, cumpliendo el objetivo del laboratorio.
3. **Consistencia sacrificada, disponibilidad priorizada:** el 61.5 % de las respuestas son datos en caché (`Cached Product A/B`), lo que refleja el trade-off explícito del patrón Fallback.
4. **Latencia acotada:** ningún `time_total` supera ~1.45 s; la ruta degradada oscila entre 0.80 y 1.45 s (suma de reintentos + delays + timeout), y la ruta exitosa permanece en milisegundos.
5. **Punto de partida para la Etapa 3 (`@CircuitBreaker`):** aunque la disponibilidad ya es 100 %, cada solicitud degradada sigue consumiendo 3 intentos y ~1 s de latencia. El `@CircuitBreaker` reducirá ese costo rechazando solicitudes sin reintentar cuando el backend esté claramente caído, llevando la latencia de las respuestas degradadas hacia el orden de milisegundos.

## 7. Evidencia entregable de la etapa

- `availability-modification2.txt` (200 líneas, formato `NNN <http_code> <time_total> <body>`).
- Salida del script con las 5 métricas agregadas.
- Fragmento de logs del servidor mostrando la secuencia `[Intento 1/3] … [Intento 2/3] … [Intento 3/3] … Fallback activado: Se agotaron todos los 3 intentos…` para las rutas degradadas, y `[Intento 1/3] Catálogo retornado exitosamente…` para las exitosas (a completar en la sección 4.4).