---
Maestro: IVONNE TATIANA ALCANTARA LLANAS
---
## 1. Datos generales

| Campo | Valor |
|---|---|
| **Laboratorio** | Clase 1.1 — Línea base de disponibilidad |
| **Servicio evaluado** | `catalog-service` (proyecto-base-unidad-01) |
| **Endpoint** | `GET http://localhost:8080/v1/products` |
| **Herramienta** | Script `test-availability.sh` (curl + bash) |
| **Solicitudes ejecutadas** | 200 |
| **Archivo de evidencia** | `availability-baseline.txt` |
| **Entorno** | Ubuntu 26.04.1 LTS, OpenJDK 25, Maven 3.9.12, Quarkus Dev Mode |

## 2. Objetivo de la práctica

Ejecutar el `catalog-service` inicial **sin patrones de resiliencia** y medir su comportamiento real bajo una secuencia de 200 solicitudes consecutivas. Esta medición constituye la **línea base** contra la cual se compararán los resultados de los laboratorios 1.2 (Timeout, Retry, Fallback, CircuitBreaker) y 1.3 (Kubernetes y autocuración).

El servicio simula intencionalmente tres escenarios probabilísticos:

- **HTTP 200 + `SUCCESS`** (50%): respuesta exitosa.
- **HTTP 500 + `RuntimeException`** (30%): falla simulada de conexión a base de datos primaria.
- **Latencia ~3000 ms** (20%): congestión simulada de base de datos.

## 3. Metodología

1. Se inició la aplicación con `./mvnw quarkus:dev`.
2. Se verificó la disponibilidad del endpoint con `curl -i http://localhost:8080/v1/products`.
3. Se creó y ejecutó el script `test-availability.sh`, que realiza 200 peticiones secuenciales capturando para cada una: **código HTTP**, **tiempo total (`time_total`)** y **cuerpo de la respuesta**.
4. Se procesaron las métricas agregadas con `grep`, `wc` y `bc`.

## 4. Resultados obtenidos

### 4.1 Métricas agregadas

| Métrica | Valor |
|---|---|
| Respuestas `SUCCESS` | **131** |
| Respuestas HTTP 200 | **131** |
| Respuestas HTTP ≠ 200 (errores 500) | **69** |
| Solicitudes totales | **200** |
| **Disponibilidad observada** | **0.6550 (65.50 %)** |

### 4.2 Salida impresa por el script

```text
Resultados guardados en: availability-baseline.txt

Respuestas SUCCESS: 131
Respuestas HTTP 200: 131
Respuestas HTTP distintas de 200: 69
Solicitudes totales: 200
Disponibilidad observada: .6550
```

### 4.3 Fragmento del archivo `availability-baseline.txt`

Se muestran solicitudes representativas de los tres escenarios simulados (éxito rápido, éxito con latencia alta y error 500). Las trazas de stack se han abreviado con `[...]` para facilitar la lectura.

```text
001 500 0.106123  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout
002 200 0.017837 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
003 200 3.005018 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
004 500 0.010306  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout
005 200 3.012608 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
006 200 0.007657 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
...
012 500 0.006292  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout
013 500 0.006214  500 - Internal Server Error [...] java.lang.RuntimeException: Database connection timeout
014 200 0.005762 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
015 200 0.003622 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
016 200 3.009226 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
...
198 200 0.002279 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
199 200 0.003453 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
200 200 0.002486 {"status":"SUCCESS","data":["Product A","Product B","Product C"]}
```
## 5. Análisis de resultados

### 5.1 Disponibilidad

La disponibilidad observada fue **65.50 %**, consistente con el rango esperado del laboratorio (60–75 %). Este valor refleja la proporción de solicitudes que retornaron HTTP 200 con cuerpo `SUCCESS`, es decir, la fracción del tráfico que el servicio atendió correctamente **sin ningún patrón de resiliencia**.

### 5.2 Distribución de errores

Los **69 errores (34.5 %)** corresponden a HTTP 500 con `RuntimeException: Database connection timeout`, disparados por la simulación probabilística de falla de BD en `CatalogResource.java:49`. La distribución observada (≈34.5 %) se aproxima al 30 % teórico, con desviación esperable por la naturaleza aleatoria de la simulación sobre una muestra de 200.

### 5.3 Latencia

Se identificaron claramente solicitudes con `time_total` cercano a **3.0 segundos** (p. ej. `003 3.005018`, `005 3.012608`, `016 3.009226`). Estas corresponden al 20 % de peticiones afectadas por la congestión simulada de BD. El resto de respuestas exitosas se resolvieron en el orden de **2–30 ms**, lo que evidencia que el problema no es de cómputo sino de **bloqueo por dependencia externa**.

### 5.4 Ausencia de `DEGRADED_CACHE`

No aparece ningún estado `DEGRADED_CACHE` en el archivo, lo cual es **correcto y esperado**: `proyecto-base-unidad-01` todavía no implementa `Fallback`. Este es precisamente el punto de partida que el laboratorio 1.2 modificará.

## 6. Conclusiones

1. **Línea base establecida:** disponibilidad del **65.50 %** y errores HTTP 500 en ~34.5 % de las solicitudes, sin mecanismos de resiliencia.
2. **Latencia crítica confirmada:** existen picos de ~3 s que, en un escenario real de ventas masivas, agotarían el pool de hilos y explicarían el colapso descrito en el escenario del laboratorio.
3. **Punto de comparación definido:** cualquier mejora introducida en 1.2 (Timeout de 800 ms, Retry, Fallback, CircuitBreaker) debe reflejarse en:
   - Aumento de la disponibilidad observable (idealmente > 95 %).
   - Desaparición de latencias de ~3 s (techo ≈ 0.8 s por `@Timeout`).
   - Aparición de respuestas `DEGRADED_CACHE` cuando el `Fallback` se active.

## 7. Evidencia entregable

- `availability-baseline.txt` (200 líneas, formato `NNN <http_code> <time_total> <body>`).
- Salida de los tres health checks (`/health`, `/ready`, `/live`).
- Este reporte con la interpretación de latencia y errores.