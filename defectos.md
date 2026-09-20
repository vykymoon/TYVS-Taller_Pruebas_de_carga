# Registro de Defectos — Taller de Pruebas de Carga

Curso: Diseño y Arquitectura de Software\
Proyecto: Pruebas de Carga y Rendimiento\
Equipo: Victor Luna, Federico Valdez y Daniel Sanabria\
Fecha: 20 de septiembre de 2026

------------------------------------------------------------------------

## Introducción

Este documento recopila los defectos identificados durante la ejecución de pruebas de rendimiento (Baseline, Load, Stress, Spike, Soak y Regresión) sobre el servicio `registraduria` (`POST /register`). Todas las cifras provienen de corridas reales de este repositorio (`perf/results/` en `master` y, para las corridas con pool, en la rama `pool`).

Las corridas se hicieron en dos computadores distintos, por lo que cada defecto indica de cuál proviene la evidencia:

-   Máquina A: AMD Ryzen 7 7700 (8 núcleos, 16 hilos), 15,9 GB de RAM, Windows 11 Home, JDK Temurin 17.0.20.
-   Máquina B: AMD Ryzen 5 5600X (6 núcleos, 12 hilos), 15,9 GB de RAM, Windows 11 Pro, JDK Temurin 25.0.4.

El detalle del ambiente está en la wiki (página "Plan de pruebas"). SLO definido: p95 ≤ 300 ms, p99 ≤ 800 ms, tasa de error < 1 %.

------------------------------------------------------------------------

## Defecto PERF-01 — Conexión JDBC nueva por operación (sin pool)

-   Capa afectada: Persistencia (`RegistryRepository.getConnection()`)
-   Escenario: Load Test (0→200 VUs, 14 min), máquina B
-   SLO definido: p95 ≤ 300 ms, error rate < 1 %
-   Resultado esperado: uso eficiente de conexiones; tiempo dentro del servidor estable bajo carga
-   Resultado obtenido: cada petición abre 2 conexiones nuevas (`existsById` + `save`). Tiempo medio dentro del servidor 2,51 ms; p95 del servidor ≈ 13,8 ms; p95 visto por k6 31,47 ms; 330 peticiones fallidas (0,003 %). El SLO se cumple, pero el diseño desperdicia capacidad.

### Evidencia

-   k6 (`summary-load-observabilidad.json`): 10.293.967 peticiones, 12.255 req/s, promedio 11,96 ms, p95 31,47 ms, máximo 909,6 ms.
-   Actuator (`actuator-prometheus.txt`): 10.293.637 peticiones exitosas, `TOTAL_TIME` 25.885,6 s → promedio 2,51 ms; p95 ≈ 13,8 ms y p99 ≈ 34,8 ms calculados con el histograma de buckets; 79,9 % de respuestas ≤ 1 ms. `actuator-requests.json` marca 10.293.639 porque incluye 2 consultas al propio Actuator.
-   `jvm.threads.live` = 212 en una lectura puntual durante la corrida (`actuator-hilos.json`).
-   Código: `DriverManager.getConnection(jdbcUrl, username, password)` en cada operación, sin pool.

### Impacto

Menor throughput y mayor latencia bajo carga concurrente que no se detectan con pruebas unitarias ni de integración. Con una base de datos real (red, autenticación, TLS) el efecto sería mayor.

### Causa probable

-   Creación y destrucción de una conexión por cada operación, dos veces por petición.

### Corrección aplicada

Se agregó HikariCP 5.1.0 en `pom.xml` y en `RegistryRepository` se creó un `HikariDataSource` con un pool de 20 conexiones (`setMaximumPoolSize(20)`); `getConnection()` pasó de `DriverManager.getConnection(...)` a `ds.getConnection()`. El cambio está en la rama `pool`. Se repitió `load` en la misma máquina (`summary-load-pool.json`, `actuator-prometheus-pool.txt`, `actuator-requests-pool.json`, `actuator-hilos-pool.json`, en la rama `pool`):

| Métrica | Sin pool | Con pool | Cambio |
|---------|----------|----------|--------|
| req/s | 12.255 | 15.367 | +25,4 % |
| p95 cliente (k6) | 31,47 ms | 21,50 ms | −31,7 % |
| Promedio servidor | 2,51 ms | 0,83 ms | −67,2 % |
| p95 servidor | ≈ 13,8 ms | ≈ 3,3 ms | −76 % |
| Errores | 330 | 0 | |

### Estado

Resuelto en la rama `pool`, validado con una corrida de `load`. Una corrida por variante: se recomienda repetirla para confirmar la magnitud de la mejora en el cliente. No se repitió el soak con pool.

### Prioridad

Media

------------------------------------------------------------------------

## Defecto PERF-02 — Fallos masivos bajo Soak (22,34 % de peticiones)

-   Capa afectada: JVM / Memoria (hipótesis, sin verificar)
-   Escenario: Soak Test (100 VUs, 2 horas), máquina A
-   SLO definido: Error rate < 1 %
-   Resultado esperado: tasa de error y latencia estables durante las 2 horas
-   Resultado obtenido: 22,34 % de peticiones fallidas (10.218.875 de 45.739.054)

### Evidencia

-   `summary-soak.json`: `http_req_failed` = 22,34 % y `register_failed` = 22,34 %; ambos umbrales (`rate<0.01`) cruzados.
-   Los umbrales de latencia se evalúan solo sobre respuestas con status 200 (p95 de 11,40 ms; 10,44 ms si se cuentan todas las peticiones), por eso el p95 y el p99 cumplen el SLO; la latencia de las respuestas fallidas no se evalúa.
-   Throughput medio de 6.353 req/s, frente a ~23.255 req/s del baseline (20 VUs) y ~18.946 req/s del load (máquina A).
-   Latencia máxima de 2.265,5 ms. El promedio de todas las peticiones (9,61 ms) supera al de las respuestas 200 (6,21 ms), lo que sugiere que las peticiones fallidas fueron más lentas, algunas cerca del timeout de 2 s del cliente.
-   Alrededor de 35,5 millones de registros exitosos acumulados en una base H2 en memoria durante la corrida.
-   El primer aviso `WARN` de k6 se observó en la consola hacia el segundo 2185 (≈ 36 min). Es una referencia aproximada, no una medición: el resumen de k6 no indica el código de estado de los fallos ni el momento exacto en que empezaron.

### Impacto

El servicio no sostiene una carga moderada y constante durante 2 horas. En un despliegue real significaría pérdida de registros y caída del servicio.

### Causa probable

-   Agotamiento del heap de la JVM por la acumulación de millones de filas en H2 en memoria (hipótesis principal).
-   Pausas largas de recolección de basura que provocan timeouts (el timeout del cliente es de 2 s).
-   Hipótesis secundaria: agotamiento de recursos por abrir una conexión nueva por operación durante mucho tiempo (sin pool). No se ha probado el soak con pool.

**Pendiente para confirmar:** revisar el log de consola del servicio de esa corrida (buscar `OutOfMemoryError` o `Java heap space`) o repetir el soak (30–40 min) midiendo `jvm.memory.used` y `jvm.gc.pause` cada minuto y registrando los códigos de estado de los fallos.

### Estado

Abierto

### Prioridad

Alta

------------------------------------------------------------------------

## Defecto PERF-03 — Fallos esporádicos de conexión bajo carga alta

-   Capa afectada: Servidor de aplicación / red local (causa sin determinar)
-   Escenario: Load sin pool (máquina B) y Stress (600 VUs, máquina A)
-   SLO definido: Error rate < 1 %
-   Resultado obtenido: 330 peticiones fallidas de 10.293.967 (0,003 %) en Load sin pool; 122 de 11.869.454 (0,001 %) en Stress

### Evidencia

-   En la corrida de Load sin pool, el servidor no registró ningún 5xx: las 10.293.637 respuestas contadas por Actuator fueron 200, y 10.293.637 + 330 = 10.293.967 peticiones enviadas por k6. Las 330 fallidas no llegaron a contarse como respuestas exitosas.
-   En la corrida de Load con pool no hubo fallos (0 de 12.908.427).
-   El resumen no desglosa el tipo de error.

### Impacto

Bajo: ambos casos están muy por debajo del 1 % permitido. Se documenta como observación.

### Causa probable

-   Errores de conexión o reinicios de conexión antes de llegar a la aplicación, posiblemente por la competencia de CPU entre k6 y el servicio, que corren en la misma máquina. No verificado.

### Estado

Abierto (observación)

### Prioridad

Baja

------------------------------------------------------------------------

## Tabla de seguimiento

| ID | Escenario | Resultado esperado | Resultado obtenido | Estado | Prioridad |
|----|-----------|--------------------|--------------------|--------|-----------|
| PERF-01 | Load (máq. B) | Uso eficiente de conexiones | 2 conexiones nuevas/petición; servidor 2,51 ms; p95 cliente 31,47 ms | Resuelto (rama `pool`) | Media |
| PERF-02 | Soak (máq. A) | Error < 1 % durante 2 h | 22,34 % de fallos | Abierto | Alta |
| PERF-03 | Load y Stress | Error < 1 % | 0,003 % (Load) y 0,001 % (Stress) | Abierto (observación) | Baja |

------------------------------------------------------------------------

## Convenciones de Estado

Abierto: Defecto identificado sin corrección aplicada.\
En progreso: En proceso de corrección.\
Resuelto: Corregido y validado con nuevas pruebas.

------------------------------------------------------------------------

Universidad de La Sabana — Facultad de Ingeniería\
Curso: Diseño y Arquitectura de Software
