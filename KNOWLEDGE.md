# Base de Conocimiento — Guía de Estudio del URL Shortener

---

## Índice

- [1. Rate Limiting L7 (NGINX)](#sec-1)
- [2. Latencia p99 — Análisis Profundo](#sec-2)
  - [Qué es realmente un percentil](#sub-2-1)
  - [Por qué el promedio está estructuralmente roto para latencia](#sub-2-2)
  - [La escalera de percentiles](#sub-2-3)
  - [El problema del "tail at scale"](#sub-2-4)
  - [Cómo se calculan los percentiles en la práctica](#sub-2-5)
  - [Histograma de Prometheus — cómo este proyecto expone la latencia](#sub-2-6)
  - [Por qué p99 < 10 ms con cache caliente](#sub-2-7)
- [3. Circuit Breaker y Degradación Elegante — Análisis Profundo](#sec-3)
  - [El problema central: fallas en cascada](#sub-3-1)
  - [La máquina de tres estados](#sub-3-2)
  - [Dos estrategias de disparo](#sub-3-3)
  - [Cómo lo implementa este proyecto (variante simplificada)](#sub-3-4)
  - [Cuándo aplicar un circuit breaker](#sub-3-5)
  - [Tradeoffs](#sub-3-6)
  - [Circuit breaker vs. patrones relacionados](#sub-3-7)
  - [Implementaciones en producción](#sub-3-8)
  - [Observabilidad para circuit breakers](#sub-3-9)
- [4. Write Concern de MongoDB — `w:majority` vs `w:1`](#sec-4)
- [5. Evicción LFU en Redis](#sec-5)
- [6. Backoff Exponencial y Thundering Herd](#sec-6)
  - [Backoff Exponencial](#sub-6-1)
  - [El problema del Thundering Herd](#sub-6-2)
- [7. Fire-and-Forget — No Esperar una Promise (Análisis Profundo)](#sec-7)
  - [La mecánica: qué pasa realmente en runtime](#sub-7-1)
  - [La implementación de este proyecto](#sub-7-2)
  - [Por qué `.catch()` no es opcional](#sub-7-3)
  - [El event loop de Node.js — lo que hace funcionar al fire-and-forget](#sub-7-4)
  - [Cuándo usar fire-and-forget](#sub-7-5)
  - [Cuándo NO usar fire-and-forget](#sub-7-6)
  - [Los peligros ocultos](#sub-7-7)
  - [Alternativas al fire-and-forget crudo](#sub-7-8)
  - [Árbol de decisión: ¿debería hacer await de esto?](#sub-7-9)
  - [Este proyecto vs. analítica de producción](#sub-7-10)
- [8. Alertas y el Pipeline de Señales SRE](#sec-8)
  - [Monitorear no es responder incidentes](#sub-8-1)
  - [El pipeline de alertas de Prometheus](#sub-8-2)
  - [`for:` — pending vs firing (debouncing)](#sub-8-3)
  - [Alertas por síntoma vs por causa](#sub-8-4)
  - [Alertas por burn-rate de SLO (el siguiente nivel)](#sub-8-5)
  - [Mecánica — qué hace Prometheus en cada ciclo](#sub-8-6)
  - [Mecánica — qué hace Alertmanager con una alerta firing](#sub-8-7)
  - [Trampa — `for:` debe ser más corto que la vida del dato en la ventana de rate](#sub-8-8)
  - [Cómo lo hace este proyecto (Alertas)](#sub-8-9)
- [9. RED y USE — Dos Métodos para Elegir Métricas](#sec-9)
  - [RED — para servicios orientados a requests (la app)](#sub-9-1)
  - [USE — para recursos (CPU, memoria, disco, pools, el event loop)](#sub-9-2)
  - [Por qué el lag del event loop es _la_ señal de saturación en Node](#sub-9-3)
  - [Cómo lo hace este proyecto (RED/USE)](#sub-9-4)
- [10. Liveness vs Readiness Probes](#sec-10)
  - [La trampa de la tormenta de reinicios](#sub-10-1)
  - [Cómo lo hace este proyecto (Probes)](#sub-10-2)
- [11. Cardinalidad de Labels de Métricas](#sec-11)
  - [Las reglas](#sub-11-1)
  - [Cómo lo hace este proyecto (Cardinalidad)](#sub-11-2)
- [12. Correlación de Logs, Métricas y Trazas](#sec-12)
  - [Cómo lo hace este proyecto (Correlación)](#sub-12-1)
- [13. Hardening de Contenedores — Radio de Impacto y Aislamiento](#sec-13)
  - [Contenedores non-root](#sub-13-1)
  - [Límites de recursos — el problema del vecino ruidoso / OOM](#sub-13-2)
  - [Comparación de secretos timing-safe](#sub-13-3)
  - [Binding de puertos solo a loopback](#sub-13-4)
  - [Cómo lo hace este proyecto (Hardening)](#sub-13-5)
- [14. Flujo de Datos de Métricas — Modelo Pull, Fuente vs Vista](#sec-14)
  - [Pull vs push](#sub-14-1)
  - [Dos familias en `/metrics`](#sub-14-2)
  - [Cómo lo hace este proyecto (Métricas)](#sub-14-3)
- [15. Parseo de URLs WHATWG y Normalización de Entradas](#sec-15)
  - [Qué es el estándar WHATWG URL](#sub-15-1)
  - [Zod `.url()` valida pero no normaliza](#sub-15-2)
  - [El bug de dedup que esto causa](#sub-15-3)
  - [El fix: `.trim()` antes de `.url()`](#sub-15-4)
  - [La regla general: normalizar entradas en el borde](#sub-15-5)
  - [Cuándo ir más lejos](#sub-15-6)
  - [Cómo lo hace este proyecto (Normalización)](#sub-15-7)
- [16. Diseño de Histogramas de Prometheus — Buckets, Labels y Cobertura de Cola](#sec-16)
  - [Por qué histogramas para latencia (no gauges, no counters)](#sub-16-1)
  - [Cómo funcionan los buckets](#sub-16-2)
  - [Diseño de buckets: cubrir tu SLO, extender la cola](#sub-16-3)
  - [El label `status_class` — segmentación por resultado sin explosión de cardinalidad](#sub-16-4)
  - [Cómo `status_class` habilita la alerta HighErrorRate](#sub-16-5)
  - [Decisiones de diseño de paneles en Grafana](#sub-16-6)
  - [Cómo lo hace este proyecto (Histogramas)](#sub-16-7)
- [17. Monolito vs Microservicios vs Arquitectura Orientada a Eventos](#sec-17)
  - [Monolito — cuándo es la decisión correcta](#sub-17-1)
  - [Microservicios — cuándo es la decisión correcta](#sub-17-2)
  - [Arquitectura orientada a eventos — cuándo es la decisión correcta](#sub-17-3)
  - [Atajo de decisión](#sub-17-4)
  - [Cómo encaja este proyecto](#sub-17-5)
- [18. API Gateway — Qué Es, Patrones, Casos de Uso](#sec-18)
  - [Qué hace realmente un API Gateway](#sub-18-1)
  - [API Gateway vs reverse proxy vs load balancer vs service mesh](#sub-18-2)
  - [Patrones centrales](#sub-18-3)
  - [Cuándo necesitás uno](#sub-18-4)
  - [Implementaciones en producción](#sub-18-5)
  - [Tradeoffs](#sub-18-6)
  - [Cómo encaja este proyecto](#sub-18-7)
- [19. SQL vs NoSQL — Qué Es Cada Uno y Cuándo Usarlos](#sec-19)
- [20. DynamoDB](#sec-20)
- [21. OpenSearch](#sec-21)
- [22. Redis a Fondo — Clave/Valor, Cache, Idempotencia y Atomicidad](#sec-22)
- [23. Pool de Conexiones](#sec-23)
- [24. Hash vs Cifrado — Simétrico y Asimétrico](#sec-24)
- [25. JWT — Autenticación, Autorización y el Chequeo `sub` == `_id`](#sec-25)
- [26. Service Mesh y el Patrón Mediator](#sec-26)
- [27. Escalado Horizontal vs Vertical](#sec-27)
- [28. Arquitectura Orientada a Eventos — Caso: Depósito Bancario](#sec-28)
- [29. Pendientes de Investigación](#sec-29)

---

<a id="sec-1"></a>

## 1. Rate Limiting L7 (NGINX)

**L7 = Capa 7 = Capa de Aplicación** del modelo de redes OSI.

El modelo OSI tiene 7 capas:

| Capa  | Nombre          | Qué ve                                   |
| ----- | --------------- | ---------------------------------------- |
| 1     | Física          | bits crudos / cables                     |
| 2     | Enlace de datos | direcciones MAC, frames                  |
| 3     | Red             | direcciones IP                           |
| 4     | Transporte      | puertos TCP/UDP                          |
| 5–6   | Sesión/Present. | (rara vez se distinguen en la práctica)  |
| **7** | **Aplicación**  | **métodos HTTP, URLs, headers, cookies** |

**El rate limiting L4** (IP + puerto) es tosco: solo sabe _quién_ se conecta.
**El rate limiting L7** es inteligente: NGINX puede inspeccionar el request HTTP y aplicar límites distintos por path, por método, por valor de header, etc.

En este proyecto, NGINX aplica **límites de rate distintos por endpoint**:

- `/api/v1/data/shorten` → 10 req/s (escritura, cara)
- `/*` redirect → 100 req/s (lectura, barata)

Esa diferenciación solo es posible en L7 — en L4 todos esos requests se ven iguales (misma IP, mismo puerto 80/443).

**Por qué importa:** el rate limiting en NGINX ocurre antes de que el request llegue a Node.js. Los requests rechazados (`429`) nunca consumen un tick del event loop. L7 nos permite proteger el path de escritura (caro) más agresivamente que el de lectura.

**Referencias:**

- [NGINX `limit_req` module docs](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html)
- [CloudFlare: What is the OSI Model?](https://www.cloudflare.com/learning/ddos/glossary/open-systems-interconnection-model-osi/)
- [NGINX blog: Rate Limiting with NGINX](https://www.nginx.com/blog/rate-limiting-nginx/)

---

<a id="sec-2"></a>

## 2. Latencia p99 — Análisis Profundo

<a id="sub-2-1"></a>

### Qué es realmente un percentil

Ordená todas las mediciones de latencia de la más rápida a la más lenta. El **percentil N** es el valor en la posición N% de esa lista ordenada.

```
10 requests, ordenados: [1, 1, 2, 2, 3, 3, 4, 5, 8, 9000] ms

p50 = valor en posición 50% → 5.º valor  = 3 ms   (la mitad más rápida, la mitad más lenta)
p90 = valor en posición 90% → 9.º valor  = 8 ms
p99 = valor en posición 99% → último 1%  = 9000 ms
avg = (1+1+2+2+3+3+4+5+8+9000) / 10      = 902.9 ms  ← completamente inútil
```

El promedio es 902 ms pero 9 de cada 10 usuarios experimentaron ≤ 8 ms. El único outlier de 9 segundos destruye el promedio. **Los promedios mienten. Los percentiles dicen la verdad.**

---

<a id="sub-2-2"></a>

### Por qué el promedio está estructuralmente roto para latencia

La media funciona bien para distribuciones simétricas (como la altura de las personas). La latencia **no es simétrica** — tiene un piso duro (no puede ser negativa) y una cola larga a la derecha (un request puede tardar arbitrariamente por una pausa de GC, contención de locks, lectura fría de disco, etc.).

```
Simétrica (la media sirve):       Latencia (la media rota):

     ▐█▌                             ▐█▌
    ▐███▌                           ▐███▌
   ▐█████▌                         ▐█████▌─────────── cola larga ──────►
  ◄──────────►                    ◄──────────────────────────────────►
     media ≈ mediana                media >> mediana
```

Esta forma de distribución se llama **sesgada a la derecha** (right-skewed). En distribuciones sesgadas, la media se corre hacia la cola. Un request de 30 segundos puede subir el promedio de 10.000 requests de 2 ms a 5 ms — escondiendo invisiblemente el outlier catastrófico.

---

<a id="sub-2-3"></a>

### La escalera de percentiles

| Percentil | En criollo                                        | Quién lo experimenta                                              |
| --------- | ------------------------------------------------- | ----------------------------------------------------------------- |
| p50       | Mediana — el usuario "típico"                     | La mitad de los usuarios                                          |
| p75       | 3 de cada 4 usuarios están al menos así de rápido | Tres cuartos de los usuarios                                      |
| p90       | 9 de cada 10 usuarios                             | Casi típico                                                       |
| p95       | 19 de cada 20 usuarios                            | Donde arrancan la mayoría de los SLAs                             |
| p99       | 99 de cada 100 usuarios                           | Donde se definen la mayoría de los SLAs de producción             |
| p999      | 999 de cada 1000 usuarios                         | 1 de cada 1000 usuarios lo sufre — a 10k req/s, son 10 usuarios/s |
| p9999     | 9999 de cada 10000 usuarios                       | Raro pero real; las pausas de GC suelen aparecer acá              |

**Elegir qué percentil te importa** depende del volumen de tráfico:

- A 1 req/s: p99 significa 1 request malo cada 100 segundos → no crítico
- A 10.000 req/s: p99 significa **100 requests malos por segundo** → se pagea a los ingenieros
- A 10.000 req/s: p999 significa **10 requests malos por segundo** → siguen siendo usuarios reales afectados

---

<a id="sub-2-4"></a>

### El problema del "tail at scale"

Acá es donde los percentiles se vuelven genuinamente sorprendentes. Imaginá que una sola llamada a backend tiene p99 = 1 ms. Bien. Ahora construís una feature que hace **100 llamadas paralelas al backend** y espera a todas.

**¿Cuál es el p99 de la respuesta combinada?**

La probabilidad de que al menos una de las 100 llamadas caiga en la cola p99:

```
P(al menos una lenta) = 1 - P(todas rápidas)
                      = 1 - (0.99)^100
                      = 1 - 0.366
                      = 63.4%
```

**El 63% de los requests de usuario van a ser lentos**, aunque cada llamada individual solo sea lenta el 1% de las veces. Cuantas más dependencias en fan-out, peor.

| Llamadas en paralelo | Probabilidad de que al menos una caiga en la cola p99 |
| -------------------- | ----------------------------------------------------- |
| 1                    | 1%                                                    |
| 10                   | 9.6%                                                  |
| 100                  | 63.4%                                                 |
| 1000                 | 99.996%                                               |

Por esto el paper de Google "The Tail at Scale" (2013) es fundacional — explica por qué **los sistemas distribuidos grandes casi siempre se sienten lentos para el usuario final** aunque cada servicio individual se vea sano en aislamiento.

**Estrategias de mitigación:**

- **Hedged requests**: mandar el mismo request a dos réplicas, usar la que responda primero, cancelar la otra
- **Presupuestos de timeout**: cada llamada downstream tiene un presupuesto de tiempo; si lo excede, usar un valor cacheado/por defecto
- **Eliminar el fan-out**: diseñar APIs que no requieran muchas llamadas paralelas por request de usuario

---

<a id="sub-2-5"></a>

### Cómo se calculan los percentiles en la práctica

**Método exacto:** ordenar todos los valores, indexar el array. Requiere guardar cada medición. Para 1 millón de requests/día = 1M de números en memoria. Impracticable a escala.

**Aproximación por histograma (lo que usa Prometheus):**

`http_duration_seconds` en este proyecto es un **histograma**. En vez de guardar cada valor, cuenta cuántas observaciones cayeron en buckets predefinidos:

```
bucket[0, 0.005]   = 9500   (requests que tardaron 0–5 ms)
bucket[0, 0.01]    = 9850   (requests que tardaron 0–10 ms, acumulado)
bucket[0, 0.025]   = 9980
bucket[0, 0.05]    = 9999
bucket[0, +Inf]    = 10000  (total)
```

Para estimar el p99: encontrar el bucket más chico cuyo límite superior acumule ≥ 99% del total. Interpolar linealmente dentro de ese bucket.

**El tradeoff:** los histogramas de Prometheus son baratos (memoria fija sin importar el tráfico) pero solo tan precisos como los límites de bucket que definas. Si tu p99 real cae entre dos límites de bucket, obtenés una estimación por interpolación lineal, no un valor exacto.

**HDR Histogram** (High Dynamic Range) es una alternativa que usa buckets logarítmicos para mantener precisión en un rango amplio (microsegundos a segundos) con muy poca memoria. Se usa en herramientas como wrk2, HdrHistogram.js.

---

<a id="sub-2-6"></a>

### Histograma de Prometheus — cómo este proyecto expone la latencia

En `src/observability/metrics.ts`, `http_duration_seconds` se define como un histograma con los límites de bucket por defecto de Prometheus. Grafana lo consulta con:

```
histogram_quantile(0.99, rate(http_duration_seconds_bucket[5m]))
```

Esto calcula el p99 sobre una ventana móvil de 5 minutos. `rate()` da las tasas por segundo de cada bucket, y `histogram_quantile` hace la interpolación.

**Qué muestra el dashboard de Grafana:**

- `histogram_quantile(0.5, ...)` → p50 (latencia mediana)
- `histogram_quantile(0.95, ...)` → p95
- `histogram_quantile(0.99, ...)` → p99 ← este es el objetivo del SLO

---

<a id="sub-2-7"></a>

### Por qué p99 < 10 ms con cache caliente

La cola en el p99 está dominada por lo que le pase al 1% más lento de los requests. Cuando Redis está caliente:

- ~99%+ de los redirects: hit en Redis → ~0.5–2 ms → nunca llegan a MongoDB
- ~1% de los redirects: miss de cache o hipo de Redis → MongoDB → ~5–20 ms

El p99 captura lo peor del camino con hit de cache más lo mejor del camino de fallback. Con cache caliente, incluso el p99 queda dentro del presupuesto de in-process + round-trip a Redis.

Cuando Redis está frío (recién reiniciado), cada request golpea MongoDB. El p99 salta a la cola de MongoDB (~20–50 ms). Este es el escenario de "degrada elegantemente" de la sección 3 — la latencia se degrada, la correctitud no.

---

**Referencias:**

- [The Tail at Scale — Google (ACM, 2013)](https://cacm.acm.org/magazines/2013/2/160173-the-tail-at-scale/fulltext)
- [Percentile latency — Brendan Gregg](https://www.brendangregg.com/FrequencyTrails/modes.html)
- [Google SRE Book: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Prometheus: Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
- [HdrHistogram — Gil Tene](http://hdrhistogram.org/)
- [Cloudflare: How we think about percentiles](https://blog.cloudflare.com/the-problem-with-averages/)

---

<a id="sec-3"></a>

## 3. Circuit Breaker y Degradación Elegante — Análisis Profundo

<a id="sub-3-1"></a>

### El problema central: fallas en cascada

Sin un circuit breaker, una dependencia que falla puede tirar abajo todo tu sistema — no por falla directa, sino por **falla en cascada**:

```
Redis se cae
  → cada llamada al cache se bloquea esperando el timeout (ej. 5s)
  → los requests de redirect se acumulan, cada uno reteniendo un thread/conexión
  → el pool de conexiones se agota
  → los requests nuevos fallan inmediatamente con "no connections available"
  → tu servicio sano ahora devuelve 500s por culpa de un cache
```

El timeout es el asesino. Si Redis tarda 5 segundos en dar timeout y tenés 1000 usuarios concurrentes, acumulaste 5000 segundos de trabajo bloqueado. El circuit breaker resuelve esto **fallando rápido** — en vez de esperar 5 segundos por llamada, devolvés `null` en microsegundos una vez que el circuito está abierto.

---

<a id="sub-3-2"></a>

### La máquina de tres estados

Un circuit breaker es literalmente una máquina de estados finitos con tres estados:

```
                   ┌──────────────────────────────────┐
                   │                                  │
              umbral                            probe exitoso
              excedido                               │
                   │                                  │
                   ▼                                  │
┌──────────┐    ┌──────┐    timeout cumplido   ┌────────────┐
│  CLOSED  │───►│ OPEN │───────────────────────► HALF-OPEN  │
│(normal)  │    │(falla│                       │(probando)  │
│          │◄───│rápido)◄──────────────────────│            │
└──────────┘    └──────┘    probe falla        └────────────┘
     ▲
     │  errores bajo el umbral → sigue closed
     └──────────────────────────────────────
```

**CLOSED (operación normal)**

- Todas las llamadas pasan a la dependencia
- Los errores se cuentan en una ventana deslizante
- Cuando la cantidad/tasa de errores excede el umbral → transición a OPEN

**OPEN (fallando rápido)**

- Todas las llamadas se rechazan inmediatamente sin tocar la dependencia
- Devuelve un valor de fallback (null, respuesta cacheada, default) o lanza inmediatamente
- Después de un timeout configurado (ej. 30s) → transición a HALF-OPEN

**HALF-OPEN (probando recuperación)**

- Se deja pasar una cantidad limitada de requests "probe"
- Si tienen éxito → transición a CLOSED (el servicio se recuperó)
- Si fallan → transición de vuelta a OPEN (reinicia el timer)

---

<a id="sub-3-3"></a>

### Dos estrategias de disparo

**Ventana por conteo:** abrir después de N fallas consecutivas.

```
errores: [ok, ok, FAIL, FAIL, FAIL, FAIL, FAIL] → umbral=5 → OPEN
```

Simple pero susceptible al ruido. Un mal lote de 5 requests dispara el breaker aunque la tasa de error global sea 0.1%.

**Ventana deslizante por tasa (preferida):** trackear la tasa de error de los últimos N segundos o N llamadas.

```
últimas 100 llamadas: 95 ok, 5 fail → tasa de error = 5%
umbral = 10% → sigue CLOSED

últimas 100 llamadas: 85 ok, 15 fail → tasa de error = 15%
umbral = 10% → OPEN
```

Más estable. Netflix Hystrix y Resilience4j usan esto. Evita dispararse por ráfagas cortas.

**Guarda de volumen mínimo de llamadas:** incluso con tasa, no abras el circuito después de 1/1 fallas (100% de error pero una sola llamada). Exigí un volumen mínimo antes de que la tasa importe:

```
if (total_calls < 20) → sigue CLOSED sin importar la tasa de error
```

---

<a id="sub-3-4"></a>

### Cómo lo implementa este proyecto (variante simplificada)

El circuit breaker de este proyecto es deliberadamente mínimo — usa los eventos del ciclo de vida de conexión de ioredis en vez de trackear tasas de error a mano:

```typescript
// UrlCache constructor
redis.on("error", () => {
  this.available = false; // circuit OPENS immediately on any error
});
redis.on("ready", () => {
  this.available = true; // circuit CLOSES when ioredis confirms reconnect
});
```

Cada operación chequea el flag antes de tocar Redis:

```typescript
async get(shortCode: string): Promise<string | null> {
  if (!this.available) return null;  // fail fast — microseconds
  try {
    return await this.redis.get(`url:${shortCode}`);
  } catch {
    return null;  // defensive: shouldn't happen when available=true, but safe
  }
}
```

**Por qué funciona acá:** ioredis ya maneja la reconexión con backoff exponencial. Emite `error` cuando un intento de conexión falla y `ready` cuando tiene éxito. Delegar el tracking de estado a ioredis evita reimplementar la lógica de reconexión. El circuito está efectivamente OPEN mientras ioredis reconecta y CLOSED cuando está listo.

**Qué se saltea esto vs. un circuit breaker completo:**

- Sin estado HALF-OPEN — ioredis prueba internamente; `available` pasa directo a `true` con `ready`
- Sin umbral por tasa — cualquier error abre (apropiado acá: Redis es binario, no degradado)
- Sin timeout por operación — el timeout de ioredis se configura a nivel cliente

---

<a id="sub-3-5"></a>

### Cuándo aplicar un circuit breaker

Aplicalo cuando **las tres** condiciones sean ciertas:

1. **La dependencia es externa** (llamada de red: base de datos, API, cache, cola). Las fallas puramente in-process no lo necesitan.
2. **El modo de falla es lento** (timeouts, no errores instantáneos). Las fallas rápidas no cascadean — las lentas sí.
3. **Existe un fallback** (dato cacheado, valor por defecto, modo degradado, encolar para después). Sin fallback, el circuit breaker solo convierte una falla lenta en una rápida — sigue siendo una falla.

**Buenos candidatos:**

- Redis / Memcached (cache — fallback: la DB)
- Procesador de pagos (fallback: encolar el intento, mostrar "procesando" al usuario)
- Servicio de email (fallback: encolar para retry)
- API de analítica de terceros (fallback: descartar el evento, loguearlo)
- Microservicio downstream (fallback: devolver respuesta cacheada/default)

**Malos candidatos (no lo necesitan):**

- Tu propio código in-process — sin red, sin riesgo de timeout
- Base de datos como fuente de verdad sin fallback — abrir el circuito solo devuelve errores igual
- Batch jobs de vida corta — el estado del circuit breaker no persiste entre corridas

---

<a id="sub-3-6"></a>

### Tradeoffs

| Preocupación      | Sin circuit breaker                          | Con circuit breaker                                               |
| ----------------- | -------------------------------------------- | ----------------------------------------------------------------- |
| Modo de falla     | Lento (los timeouts se acumulan)             | Rápido (rechazo inmediato)                                        |
| Recuperación      | Automática cuando la dependencia se recupera | Requiere ciclo de probe HALF-OPEN (agrega una demora)             |
| Requiere fallback | No (pero vas a devolver errores igual)       | Sí — hay que definir qué devolver cuando está abierto             |
| Complejidad       | Baja                                         | Moderada (máquina de estados, umbrales, timers)                   |
| Falsos positivos  | N/A                                          | Puede abrirse por picos transitorios, rechazando requests válidos |
| Observabilidad    | Fácil (solo mirar errores)                   | Hay que instrumentar las transiciones de estado                   |

**El problema del falso positivo:** si tu umbral de error es demasiado agresivo, un hipo breve de red abre el circuito y empezás a rechazar requests sanos innecesariamente. Soluciones:

- Subir el volumen mínimo de llamadas antes de disparar
- Usar una ventana deslizante más larga
- Usar HALF-OPEN con incremento gradual de tráfico (probe canario) en vez de pasa/no-pasa binario

---

<a id="sub-3-7"></a>

### Circuit breaker vs. patrones relacionados

Se confunden seguido. Resuelven problemas distintos y se usan frecuentemente juntos:

| Patrón                | Qué hace                                                 | Cuándo usarlo                                      |
| --------------------- | -------------------------------------------------------- | -------------------------------------------------- |
| **Circuit breaker**   | Deja de llamar a una dependencia que falla; falla rápido | La dependencia está lenta/caída                    |
| **Retry con backoff** | Reintenta después de una falla                           | Errores transitorios (blip momentáneo)             |
| **Timeout**           | Abandona una llamada que tarda demasiado                 | Prevenir esperas sin límite                        |
| **Bulkhead**          | Aísla pools de threads/conexiones por dependencia        | Una dependencia lenta no debe bloquear a las demás |
| **Fallback**          | Devuelve datos alternativos cuando el primario falla     | Cualquier falla que necesite respuesta             |

**Combinándolos correctamente:**

```
Request
  → Timeout (no esperar para siempre)
      → Circuit breaker (no intentar si ya está fallando)
          → Retry con backoff (probar algunas veces para errores transitorios)
              → Bulkhead (limitar llamadas concurrentes a esta dependencia)
                  → Llamada real
                      → Fallback (devolver default si todo lo demás falla)
```

Retry sin circuit breaker es peligroso: 100 clientes reintentando 3 veces cada uno = carga amplificada 300× sobre un servicio que ya está sufriendo.

---

<a id="sub-3-8"></a>

### Implementaciones en producción

| Librería                                                           | Lenguaje | Notas                                                            |
| ------------------------------------------------------------------ | -------- | ---------------------------------------------------------------- |
| [opossum](https://nodeshift.dev/opossum/)                          | Node.js  | La más popular; máquina de estados completa, métricas, eventos   |
| [Resilience4j](https://resilience4j.readme.io/docs/circuitbreaker) | Java     | Sucesor de Hystrix; ventanas por conteo + tasa                   |
| [Polly](https://github.com/App-vNext/Polly)                        | .NET     | Composición de políticas: retry + CB + timeout + bulkhead        |
| [Hystrix](https://github.com/Netflix/Hystrix)                      | Java     | El original de Netflix; en modo mantenimiento, usar Resilience4j |
| [pybreaker](https://github.com/danielfm/pybreaker)                 | Python   | Simple, usado en producción                                      |

**Ejemplo con opossum** (cómo se ve un circuit breaker completo en Node.js):

```typescript
import CircuitBreaker from "opossum";

const breaker = new CircuitBreaker(redisGet, {
  timeout: 3000, // 3s → timeout = OPEN
  errorThresholdPercentage: 50, // 50% error rate → OPEN
  resetTimeout: 30000, // 30s in OPEN before HALF-OPEN probe
  volumeThreshold: 10, // minimum 10 calls before rate is evaluated
});

breaker.fallback(() => null); // return null when OPEN (same as cache miss)

breaker.on("open", () => logger.warn("circuit opened"));
breaker.on("halfOpen", () => logger.info("circuit probing"));
breaker.on("close", () => logger.info("circuit closed"));
```

Este proyecto usa una implementación manual en su lugar — más simple y suficiente para una dependencia binaria como Redis.

---

<a id="sub-3-9"></a>

### Observabilidad para circuit breakers

Un circuit breaker que se abre en silencio es peligroso — podrías no enterarte de que corrés sin cache durante horas. Instrumentá:

1. **Transiciones de estado** → log + counter en OPEN/HALF-OPEN/CLOSE
2. **Tasa de rechazo** → cuántas llamadas se cortocircuitaron por segundo
3. **Tasa de invocación del fallback** → qué porcentaje de requests usó el fallback
4. **Tasa de error por dependencia** → desglose por servicio para aislar cuál está fallando

En este proyecto, `redis_errors_total` en Prometheus cubre esto parcialmente. Una implementación completa también exportaría un gauge `cache_circuit_state` (0=closed, 1=open, 2=half-open).

---

**Referencias:**

- [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Release It! — Michael Nygard (libro)](https://pragprog.com/titles/mnee2/release-it-second-edition/) — acuñó el término en software
- [AWS: Circuit Breaker pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/circuit-breaker.html)
- [Netflix Tech Blog: Making Netflix API More Resilient](https://netflixtechblog.com/making-the-netflix-api-more-resilient-a8ec62159c2d)
- [Resilience4j: Circuit Breaker docs](https://resilience4j.readme.io/docs/circuitbreaker)
- [opossum: Node.js circuit breaker](https://nodeshift.dev/opossum/)

---

<a id="sec-4"></a>

## 4. Write Concern de MongoDB — `w:majority` vs `w:1`

MongoDB puede correr como **replica set**: múltiples servidores que guardan copias de los datos (1 primario + N secundarios). Las escrituras van al primario; los secundarios replican de forma asíncrona.

El **write concern** controla cuándo MongoDB confirma (acknowledge) una escritura al cliente:

| Write Concern | Se confirma cuando…                                     | Riesgo si el primario crashea                                                              |
| ------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `w:1`         | El primario escribió en memoria (aún no replicado)      | Se pueden perder datos (el primario crasheó antes de que los secundarios se pongan al día) |
| `w:majority`  | Una mayoría de los miembros del replica set escribieron | El dato es durable — aunque el primario crashee, los secundarios lo tienen                 |

**En este proyecto:**

- `mainClient` usa `w:majority` — los registros de URL y el contador de IDs deben ser durables. Perder el contador causaría colisiones de ID en el próximo reset de secuencia.
- `analyticsClient` usa `w:1` — los eventos de click son best-effort. Perder un registro de click en un crash es aceptable. El write concern más bajo = menor latencia en el path de escritura de analítica.

**Analogía simple:** `w:majority` es como un escribano que manda una copia a tres oficinas antes de firmar. `w:1` es como el escribano firmando inmediatamente y mandando las copias después.

**Referencias:**

- [MongoDB: Write Concern docs](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [MongoDB: Replica Set Write Concern](https://www.mongodb.com/docs/manual/core/replica-set-write-concern/)

---

<a id="sec-5"></a>

## 5. Evicción LFU en Redis

Cuando Redis se queda sin memoria, tiene que evictar (borrar) algunas claves para hacer lugar a las nuevas. La **política de evicción** decide _qué_ claves borrar.

**LFU = Least Frequently Used** (menos frecuentemente usada). Redis trackea con qué frecuencia se accedió cada clave. Cuando la memoria se llena, evicta las claves con la _menor frecuencia de acceso_ — las que se usaron menos veces.

**Otras políticas de evicción para comparar:**

| Política         | Evicta…                                                              |
| ---------------- | -------------------------------------------------------------------- |
| `noeviction`     | Nada — devuelve error en escrituras nuevas cuando está lleno         |
| `allkeys-lru`    | Least Recently Used — la clave inactiva hace más tiempo              |
| `allkeys-lfu`    | Least Frequently Used — la clave accedida menos veces (la usada acá) |
| `allkeys-random` | Una clave al azar                                                    |
| `volatile-*`     | Igual que las anteriores pero solo entre claves con TTL configurado  |

**¿Por qué LFU sobre LRU para este caso de uso?**

El acceso a URLs sigue una **ley de potencia**: ~20% de las URLs reciben ~80% del tráfico (pensá: un tweet viral vs. un link de una sola vez). Con LRU, una URL popular que no fue accedida en la última hora podría ser evictada aunque se pida millones de veces por día. LFU mantiene las claves de alta frecuencia incluso durante períodos tranquilos.

Ejemplo:

- Clave A: accedida 10.000 veces esta semana, sin accesos en los últimos 30 min
- Clave B: accedida 1 vez, hace 5 minutos

LRU evicta A (último acceso más viejo). LFU evicta B (menor frecuencia total). LFU es lo correcto acá.

**Internals de LFU en Redis:** Redis no guarda conteos exactos. Usa un contador probabilístico llamado **contador de Morris** (aproximación logarítmica) que entra en 8 bits. Esto mantiene el overhead mínimo.

**Referencias:**

- [Redis: Using Redis as an LFU cache](https://redis.io/docs/manual/eviction/#using-redis-as-an-lfu-cache)
- [Redis eviction policies — lista completa](https://redis.io/docs/manual/eviction/)
- [Wikipedia: Least Frequently Used](https://en.wikipedia.org/wiki/Least_frequently_used)

---

<a id="sec-6"></a>

## 6. Backoff Exponencial y Thundering Herd

<a id="sub-6-1"></a>

### Backoff Exponencial

Cuando una conexión falla, el cliente espera antes de reintentar. **Backoff exponencial** significa que el tiempo de espera se duplica con cada intento fallido:

```
intento 1 → espera 100ms
intento 2 → espera 200ms
intento 3 → espera 400ms
intento 4 → espera 800ms
...
intento N → espera min(100ms × 2^N, 30s)
```

El tope `min(..., 30s)` evita que la espera crezca hasta el infinito. Pasado cierto punto, no tiene sentido esperar más de 30 segundos.

**¿Por qué no reintentar inmediatamente?** Si algo está caído, martillarlo con retries cada milisegundo empeora el problema. Los retries rápidos:

1. Desperdician CPU/red de ambos lados
2. Pueden impedir que el servicio caído se recupere (está ocupado atendiendo tormentas de retries)

<a id="sub-6-2"></a>

### El problema del Thundering Herd

Imaginá 500 instancias de la app conectadas a Redis. Redis crashea y reinicia 10 segundos después.

**Sin backoff:** las 500 instancias detectan la reconexión casi en el mismo momento y disparan sus requests de conexión simultáneamente. Redis, recién arrancando, recibe 500 conexiones de golpe — potencialmente crasheando de nuevo o poniéndose muy lento.

**Con backoff exponencial + jitter:** cada instancia esperó un tiempo distinto antes de reintentar (porque el backoff está escalonado y el jitter agrega aleatoriedad). Las reconexiones llegan distribuidas a lo largo de varios segundos — Redis las maneja cómodamente.

La fórmula de backoff en este proyecto:

```
delay = min(100ms × 2^attempts, 30_000ms)
```

El **jitter** (varianza aleatoria agregada al delay) es una mejora común que distribuye aún más los intentos de reconexión en el cluster. ioredis agrega un poco automáticamente.

**Referencias:**

- [AWS: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Wikipedia: Thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem)
- [ioredis retry strategy docs](https://github.com/redis/ioredis#auto-reconnect)

---

<a id="sec-7"></a>

## 7. Fire-and-Forget — No Esperar una Promise (Análisis Profundo)

<a id="sub-7-1"></a>

### La mecánica: qué pasa realmente en runtime

En JavaScript/TypeScript, las operaciones async devuelven **Promises**. Hay dos formas de llamarlas:

**Con await:**

```typescript
const result = await doSomething(); // current function suspends here
res.json({ ok: true }); // runs AFTER doSomething() resolves
```

**Sin await (fire-and-forget):**

```typescript
doSomething(); // Promise created, I/O submitted to event loop
res.json({ ok: true }); // runs IMMEDIATELY on next line — no wait
// doSomething() settles later, on a future event loop tick
```

La clave: `await` **no** bloquea el thread. Suspende la función `async` actual y devuelve el control al event loop, que corre otros callbacks. Cuando la Promise se resuelve, la función se reanuda. No hacer await se saltea la suspensión por completo — la función actual nunca se pausa.

#### Traza exacta de `await` — qué significa "suspender" paso a paso

Es tentador pensar "el event loop corre otras funciones mientras esta espera, y después vuelve a ella". Cerca, pero al revés en un punto clave: nada intercala activamente con la función pausada. Se suspende y devuelve el control **inmediatamente**; el event loop solo agarra otro trabajo porque el stack quedó vacío — no porque esté haciendo malabares con esta función y otras.

```typescript
async function handler(req, res) {
  const result = await doSomething(); // (A)
  res.json({ ok: true }); // (B)
}
```

1. `handler` corre sincrónicamente hasta `doSomething()`.
2. `doSomething()` se ejecuta — si hace I/O (ej. una query a Mongo), le entrega esa operación a libuv/el SO y devuelve una **Promise pendiente**.
3. `await` ve una Promise pendiente → **suspende `handler` ahí mismo**. Todo lo que sigue (línea B) se vuelve una continuación — efectivamente un `.then()` implícito sobre esa Promise.
4. `handler` devuelve el control — su propia Promise (toda función `async` devuelve una) queda pendiente. El call stack se vacía.
5. Con el stack vacío, el event loop atiende lo que esté listo: otro request entrante, un timer, otra Promise resuelta. Esta es la parte de "corre otro trabajo" — pasa porque el stack está libre, no porque Node esté repartiendo tiempo activamente entre `handler` y otra cosa.
6. Cuando Mongo responde, libuv encola el callback correspondiente en la cola de I/O pendiente.
7. En un tick futuro, ese callback corre y **resuelve** la Promise de `doSomething()`.
8. Resolverla **agenda la continuación de `handler` como microtask** (línea B, con `result` ya asignado) — no reanuda `handler` sincrónicamente dentro de ese mismo callback.
9. Cuando el stack se vuelve a vaciar, la cola de microtasks se drena → `handler` se reanuda exactamente en la línea B.

**Dos requests concurrentes, para hacer concreto el punto de "sin paralelismo real":**

```
Request 1 llega → handler1 corre hasta su await → se suspende, stack vacío
Request 2 llega → handler2 corre hasta su await → se suspende, stack vacío
...pasa el tiempo, Mongo responde al request 1...
libuv resuelve promise1 → microtask encolada → handler1 se reanuda en línea B → res.json() para request 1
...Mongo responde al request 2...
libuv resuelve promise2 → microtask encolada → handler2 se reanuda en línea B → res.json() para request 2
```

En ningún momento hay dos funciones corriendo "a la vez" — una siempre corre hasta que se suspende (`await`) o termina, y entonces el thread pasa a lo que siga en la fila.

**Concepto erróneo común a evitar:** no es que "libuv resuelve el `await`". Libuv resuelve el I/O de bajo nivel (la lectura/escritura del socket TCP) — eso dispara un callback que resuelve la Promise de JS, y _esa_ resolución es lo que agenda la continuación de la función async como microtask. Tres pasos distintos, no uno.

**Esquema del event loop de Node.js:**

```
Tick 1:  Handle incoming HTTP request
           → call redirect controller
           → send HTTP response  ✓
           → submit MongoDB insert to I/O queue (not awaited)
           → controller returns

Tick 2–N: Other requests handled

Tick N+1: MongoDB insert I/O completes
           → .catch() callback runs if error
           → done
```

La respuesta ya se fue para el tick N+1. El insert y la respuesta son **concurrentes**, no secuenciales.

---

<a id="sub-7-2"></a>

### La implementación de este proyecto

```typescript
// redirect.controller.ts
res.redirect(config.redirectCode, longUrl); // HTTP 302 sent to client NOW

clickRepo
  .insert({ shortUrl, timestamp, ip, userAgent, referrer })
  .catch((err) => {
    req.log.warn({ err }, "click write failed");
    clickWriteErrors.inc(); // Prometheus counter — makes silence visible
  });
// controller returns. insert is still running in background.
```

Tres cosas pasando acá:

1. Respuesta enviada — el navegador del usuario empieza a seguir el redirect
2. El insert de MongoDB arranca — async, no bloqueante
3. `.catch()` registrado — los errores aparecen en logs y métricas, nunca se tragan

---

<a id="sub-7-3"></a>

### Por qué `.catch()` no es opcional

Una Promise sin await y sin handler de error que rechaza emite `unhandledRejection` en el proceso de Node.js. En Node 15+, esto **termina el proceso por defecto**. En versiones anteriores es un warning, pero el default eventualmente será crash en todos lados.

```typescript
// WRONG — if insert() throws, process crashes or warns
clickRepo.insert({ ... });

// CORRECT — error handled, process stays up
clickRepo.insert({ ... }).catch((err) => logger.warn({ err }, "click failed"));

// ALSO CORRECT — explicit void signals intentional fire-and-forget to linters
void clickRepo.insert({ ... }).catch((err) => logger.warn({ err }, "click failed"));
```

TypeScript con `@typescript-eslint/no-floating-promises` marca las Promises sin await que no tengan `void` — el operador `void` señala explícitamente "sé que esto no tiene await y estoy de acuerdo con eso".

---

<a id="sub-7-4"></a>

### El event loop de Node.js — lo que hace funcionar al fire-and-forget

Node.js es single-threaded pero no bloqueante. Todo el I/O (red, disco, timers) lo maneja **libuv** — una librería en C que administra un thread pool y el I/O asíncrono a nivel de SO por debajo. El thread de JS de Node nunca se bloquea en I/O; solo envía trabajo y registra callbacks.

**Fases del event loop (simplificado):**

```
┌─────────────────────────────────────────────┐
│              Event Loop Tick                │
│                                             │
│  1. timers       (setTimeout, setInterval)  │
│  2. pending I/O  (completed I/O callbacks)  │
│  3. idle/prepare (internal)                 │
│  4. poll         (wait for new I/O events)  │
│  5. check        (setImmediate callbacks)   │
│  6. close        (socket close events)      │
└─────────────────────────────────────────────┘
         ↑                          ↓
         └──────────── repeat ──────┘
```

Cuando llamás `clickRepo.insert(...)`, Node envía una escritura TCP al socket de MongoDB (manejado por libuv) e inmediatamente devuelve una Promise. El thread de JS sigue. Cuando MongoDB confirma la escritura, libuv pone el callback en la cola de "pending I/O". En el próximo tick del event loop, el callback corre — lo cual resuelve o rechaza la Promise.

**Por esto fire-and-forget funciona en Node pero es peligroso en threads de Go/Java:** en runtimes multi-threaded, "trabajo en background" significa spawnear un thread. La creación ilimitada de threads puede agotar recursos. En Node, el event loop es el scheduler — no hay costo de thread por agregar otro callback de I/O pendiente.

#### Microtasks vs. macrotasks (dónde corren realmente las Promises)

Las 6 fases de arriba son **macrotasks**. Las Promises (`.then`, `.catch`, y todo lo que sigue a un `await`) no esperan a la próxima fase — corren en la **cola de microtasks**, que se drena por completo **entre cada callback individual**, no solo entre fases.

```
Call stack vacío
  → drenar cola de microtasks (TODA, incluyendo microtasks nuevas encoladas durante el drenado)
  → correr UNA macrotask (ej. un callback de timer, un callback de I/O)
  → drenar cola de microtasks de nuevo
  → correr la próxima macrotask
  → ...
```

Orden de prioridad cuando ambas están pendientes: primero la cola de `process.nextTick()` (específica de Node, se drena por completo antes que las microtasks), después la cola de microtasks de Promises, después la próxima macrotask/fase. Por esto:

```typescript
setTimeout(() => console.log("timeout"), 0); // macrotask — phase: timers
Promise.resolve().then(() => console.log("promise")); // microtask
process.nextTick(() => console.log("nextTick")); // nextTick queue
// Output: nextTick, promise, timeout — always, regardless of the 0ms delay
```

**Consecuencia práctica para este codebase:** `clickRepo.insert(data).catch(...)` — el callback de `.catch` es una microtask. No corre a mitad de fase intercalado con otras macrotasks; corre en el instante en que el stack se libera después de que dispara el callback del I/O subyacente, antes de que Node pase al próximo timer/evento de socket agendado. Un loop `while(true)` de scheduling sincrónico de microtasks (ej. llamar recursivamente `Promise.resolve().then(fn)`) puede por lo tanto matar de hambre a las macrotasks (timers, sockets HTTP entrantes) indefinidamente — un footgun real de Node distinto del caso de bloqueo de CPU en la sección `USE` de más abajo.

**El thread pool de libuv — no todo es verdaderamente async:** los sockets de red (HTTP, MongoDB, Redis sobre TCP) usan el I/O asíncrono nativo del SO (epoll/kqueue/IOCP) — genuinamente sin thread involucrado. Pero `fs.*`, `dns.lookup`, `crypto.pbkdf2` y `zlib` están atados a CPU/syscalls bloqueantes por debajo, así que libuv los corre en un pequeño pool de worker threads (tamaño por defecto **4**, ajustable vía `UV_THREADPOOL_SIZE`). Saturá ese pool (ej. 10 llamadas concurrentes a `bcrypt.hash()`) y la 5.ª llamada queda en cola detrás de las primeras 4, aunque tu código JS nunca se bloquee — una sorpresa común cuando se razona "Node es async así que nada hace cola".

---

<a id="sub-7-5"></a>

### Cuándo usar fire-and-forget

Usalo cuando **las tres** condiciones se cumplan:

1. **El caller no necesita el resultado.** Nada de lo que le devolvés al usuario depende de que esta operación complete.
2. **La falla es aceptable (semántica best-effort).** Perder la operación no tiene impacto en correctitud — solo en observabilidad.
3. **El trabajo está acotado.** No estás encolando tareas infinitas que se acumulan sin límite si una dependencia está lenta.

**Buenos casos de uso:**

| Caso de uso                          | Por qué F&F encaja                                               |
| ------------------------------------ | ---------------------------------------------------------------- |
| Analítica de clicks/vistas           | El usuario no necesita esperar; perder un click es aceptable     |
| Escrituras de audit log              | Observabilidad, no correctitud; no bloquear el flujo del usuario |
| Calentar cache después de leer la DB | Si falla, el próximo request solo tiene un cache miss            |
| Enviar un email de bienvenida        | La entrega de email es async igual; falla → cola de retry        |
| Actualizar un timestamp "last seen"  | Un dato levemente viejo está bien                                |
| Incrementar un counter de Prometheus | En memoria; no puede fallar en el sentido tradicional            |
| Entrega de webhooks a terceros       | Mandar y seguir; la entrega es problema de ellos                 |

---

<a id="sub-7-6"></a>

### Cuándo NO usar fire-and-forget

**Nunca hagas fire-and-forget cuando:**

1. **La correctitud requiere la escritura.** Si el dato debe existir antes de responder, tenés que hacer await.

   ```typescript
   // WRONG: user gets shortUrl but DB write might not have happened yet
   urlRepo.insert(urlDoc); // not awaited!
   return res.json({ shortUrl });
   ```

2. **El caller necesita el resultado.** Cualquier patrón `await result =` significa que necesitás el valor de retorno.

3. **Necesitás consistencia transaccional.** Si A y B deben tener éxito o fallar juntos, fire-and-forget rompe la atomicidad.

4. **El agotamiento de recursos es posible.** Si el trabajo en background es lento y los requests se acumulan, podés encolar miles de inserts pendientes en memoria sin backpressure.

   ```typescript
   // HIGH TRAFFIC: 10k req/s, MongoDB slow → 10k pending inserts in event loop queue
   // Memory grows; no backpressure; eventual OOM
   heavyWork(); // not awaited, repeated at high rate
   ```

5. **El graceful shutdown necesita drenar.** Si tu proceso recibe SIGTERM con 500 inserts fire-and-forget en vuelo, todos mueren en silencio.

---

<a id="sub-7-7"></a>

### Los peligros ocultos

#### 1. Pérdida silenciosa de datos en el shutdown

```typescript
// SIGTERM arrives while 200 inserts are pending
process.on("SIGTERM", async () => {
  await server.close();
  await mongo.close();
  process.exit(0); // ← the 200 in-flight inserts are GONE
});
```

Soluciones:

- **Contador de drenado:** incrementar un contador antes de cada F&F, decrementar en `.finally()`. En el shutdown, esperar hasta que el contador llegue a 0.
- **Cola de mensajes:** en vez de escribir directo, pushear a una cola (Redis, RabbitMQ, SQS). La cola persiste entre crashes. Un worker lee e inserta.
- **Aceptar la pérdida:** documentarla explícitamente y monitorear con `clickWriteErrors` — el enfoque de este proyecto. Correcto para analítica; incorrecto para datos financieros.

#### 2. Concurrencia sin control / backpressure

Cada Promise sin await es una tarea en el event loop sin límite. Si tu downstream está lento:

```
10,000 requests/s → 10,000 inserts concurrentes a la DB
MongoDB maneja 1,000 inserts/s → la cola crece a 9,000/s
Después de 1 minuto: 540,000 inserts pendientes en memoria
→ crash por OOM
```

Solución: usar un semáforo o una cola acotada.

```typescript
import pLimit from "p-limit";
const limit = pLimit(100); // max 100 concurrent

// instead of bare fire-and-forget:
limit(() => clickRepo.insert(data)).catch(handleErr);
```

#### 3. Fuga de contexto/scope

Las variables capturadas por el closure del F&F quedan en memoria hasta que la Promise se resuelva. Si el closure captura un objeto request grande o una conexión de base de datos, esa memoria no puede ser recolectada por el GC.

```typescript
// req is captured in closure — stays alive until insert settles
const { ip, userAgent, referrer } = req; // extract primitives instead
clickRepo.insert({ ip, userAgent, referrer }).catch(logger.warn);
```

Siempre extraé valores primitivos de los objetos request antes de disparar; no cierres sobre el `req` completo.

---

<a id="sub-7-8"></a>

### Alternativas al fire-and-forget crudo

| Alternativa                                         | Cuándo usarla                                                                  | Tradeoff                                           |
| --------------------------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------- |
| **Await**                                           | Se necesita el resultado o la falla es inaceptable                             | Agrega latencia a la respuesta                     |
| **Cola de mensajes** (Redis Streams, SQS, RabbitMQ) | Se requiere durabilidad, alto volumen, se necesita retry                       | Agrega infraestructura; sobrevive crashes          |
| **`Promise.allSettled()`**                          | Querés correr múltiples tareas y responder cuando todas terminen, fallen o no  | Sigue siendo awaited — agrega la duración completa |
| **`setImmediate()`**                                | Diferir trabajo sync de CPU para después del I/O actual, no para trabajo async | No ayuda con I/O async                             |
| **Worker threads**                                  | Trabajo pesado atado a CPU                                                     | Thread separado, overhead de copia de memoria      |
| **p-limit / bottleneck**                            | F&F pero con tope de concurrencia (backpressure)                               | Overhead chico; previene OOM bajo carga            |

**Patrón de cola de mensajes** (la versión production-grade de la analítica de este proyecto):

```typescript
// Instead of:
clickRepo.insert(data).catch(logger.warn);

// Durable version:
await redisStream.xadd("clicks", "*", data);
// A separate worker process reads the stream and inserts to MongoDB
// Survives crashes, has retry logic, naturally backpressures
```

#### Cómo funciona realmente `XADD`

`XADD key ID field value [field value ...]` agrega una entrada a un Redis Stream — un tipo de dato de log append-only (no es pub/sub; las entradas persisten hasta que se recorten).

- `key`: nombre del stream (`"clicks"`).
- `ID`: `"*"` le dice a Redis que lo autogenere, formato `<ms-timestamp>-<seq>` (ej. `1719840000123-0`). Los IDs son estrictamente crecientes, así que sirven también como offset/cursor durable.
- Los fields se guardan como un mapa plano en esa entrada (como un mini hash), sin esquema.
- El stream vive en la memoria de Redis. Con persistencia AOF (Append Only File → seguridad de datos) o RDB (Redis Database → backups chicos, restart rápido a costa de perder un poco de data) habilitada sobrevive un restart de Redis; sin ella, un crash de Redis pierde las entradas no consumidas (el mismo techo de durabilidad que cualquier dato de Redis — por esto algunos equipos ponen Kafka/SQS delante de cualquier cosa crítica para facturación).
- Los streams sin límite crecen para siempre — recortalos con `MAXLEN ~ 100000` en el `XADD` (`~` = recorte aproximado, más barato) o un `XTRIM` separado.

El lado consumidor (el "worker process separado" mencionado arriba) usa un **consumer group**, no una lectura simple, para que el trabajo sea durable y balanceado:

```typescript
// One-time setup: creates the consumer group "click-writers" on stream "clicks".
// "$" = start from entries added after this point (ignore backlog); MKSTREAM = create the stream if missing.
// This does NOT create a "consumer" — consumers are just names workers pick when they call xreadgroup below.
await redis.xgroup("CREATE", "clicks", "click-writers", "$", "MKSTREAM");

// Worker loop: long-running process, never exits — poll forever, same shape as any queue consumer (SQS, Kafka, etc).
while (true) {
  const res = await redis.xreadgroup(
    "GROUP",
    "click-writers", // read as a member of this consumer group
    "worker-1", // this consumer's name — run more workers as "worker-2", "worker-3"... for load balancing
    "COUNT",
    100, // max entries to grab per call
    "BLOCK",
    5000, // wait up to 5s for new entries instead of busy-looping when the stream is empty
    "STREAMS",
    "clicks",
    ">", // ">" = only entries never delivered to this group before (not a specific ID/backlog replay)
  );
  // res shape: [["clicks", [[id, fields], [id, fields], ...]]] — one entry per stream requested (just "clicks" here).
  // res?.[0]?.messages is that stream's batch of [id, fields] pairs; ?? [] guards the BLOCK timeout (no new data → empty).
  for (const [id, fields] of res?.[0]?.messages ?? []) {
    await clickRepo.insert(parseFields(fields));
    await redis.xack("clicks", "click-writers", id); // mark done — removes it from this group's pending list (PEL)
  }
}
```

- `XACK` saca la entrada de la **pending entries list (PEL)** del grupo. Si el worker crashea antes de ackear, la entrada queda en la PEL.
- `XPENDING clicks click-writers` inspecciona entradas trabadas/sin ack; `XCLAIM` permite que otro consumidor robe una entrada inactiva por más de N ms (recupera de un worker muerto) — esta es la lógica de retry referida arriba.
- El backpressure es natural: el stream es solo una lista en memoria, así que un consumidor lento hace crecer la lista, pero el productor (`XADD`) nunca se bloquea por el consumidor — sigue siendo fire-and-forget desde la perspectiva de la API, solo que ahora durable.

#### Alternativa: AWS SQS

Mismo objetivo de "desacoplar productor de consumidor", tradeoffs distintos — SQS es una cola administrada con salto de red, en vez de una estructura de Redis casi in-process.

```typescript
// Producer (in the request path, still not awaited-for-correctness — just durable)
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";
const sqs = new SQSClient({ region: "us-east-1" });

await sqs.send(
  new SendMessageCommand({
    QueueUrl: process.env.CLICKS_QUEUE_URL,
    MessageBody: JSON.stringify(data),
  }),
);
```

```typescript
// Consumer — separate worker process/Lambda, long-polling
import {
  ReceiveMessageCommand,
  DeleteMessageCommand,
} from "@aws-sdk/client-sqs";

while (true) {
  const { Messages } = await sqs.send(
    new ReceiveMessageCommand({
      QueueUrl: process.env.CLICKS_QUEUE_URL,
      MaxNumberOfMessages: 10,
      WaitTimeSeconds: 20, // long poll — avoids busy-loop cost
      VisibilityTimeout: 30, // hidden from other consumers while being processed
    }),
  );
  for (const msg of Messages ?? []) {
    await clickRepo.insert(JSON.parse(msg.Body));
    await sqs.send(
      new DeleteMessageCommand({
        QueueUrl: process.env.CLICKS_QUEUE_URL,
        ReceiptHandle: msg.ReceiptHandle, // ack equivalent
      }),
    );
  }
}
```

- `VisibilityTimeout` es la respuesta de SQS a `XCLAIM`: si el consumidor no hace `DeleteMessage` (ack) dentro del timeout, el mensaje vuelve a ser visible para otro consumidor — automático, sin inspección manual de pending lists.
- Configurá una **redrive policy** hacia una Dead-Letter Queue (DLQ) después de N recepciones fallidas — el equivalente built-in de trackear manualmente entradas de `XPENDING` que nunca se ackean.
- Colas Standard: entrega at-least-once, orden best-effort, throughput casi ilimitado. Colas FIFO: procesamiento exactly-once + orden estricto por `MessageGroupId`, con tope de 3.000 msg/s (con batching).
- Sin infraestructura propia que correr — AWS administra almacenamiento/escalado/durabilidad (replicación multi-AZ). A cambio: costo por request, latencia de red por llamada (vs. Redis usualmente co-ubicado), y una dependencia dura de que AWS esté arriba.

|                 | Redis Streams                                                          | AWS SQS                                    | Kafka                                            |
| --------------- | ---------------------------------------------------------------------- | ------------------------------------------ | ------------------------------------------------ |
| Carga operativa | Vos corrés/escalás Redis                                               | Cero — totalmente administrado             | Vos corrés/escalás brokers (o MSK)               |
| Latencia        | Sub-ms, dentro de la VPC                                               | ~10-20ms por llamada API                   | Sub-ms a pocos ms                                |
| Durabilidad     | Solo tan durable como tu config de persistencia de Redis               | Durable por defecto (multi-AZ)             | Durable, replicado, retención larga              |
| Orden           | Por stream, estricto                                                   | Solo colas FIFO, por grupo                 | Por partición, estricto                          |
| Modelo de retry | Manual (`XPENDING`/`XCLAIM`)                                           | Automático (visibility timeout + DLQ)      | Manual (manejo de offsets del consumidor)        |
| Encaje acá      | El más barato si Redis ya está en el stack (lo está, en este proyecto) | Bueno si ya estás en AWS y querés cero ops | Overkill salvo que volumen/replay lo justifiquen |

#### Qué es una Dead-Letter Queue (DLQ)

Una DLQ es una cola secundaria que recibe los mensajes que un consumidor **no pudo procesar exitosamente después de N intentos**, en vez de que esos mensajes reintenten para siempre o se descarten en silencio.

Por qué existe: sin una, un mensaje veneno (payload malformado, un bug que lanza en cada retry, una fila que viola una constraint de la DB) o bien (a) bloquea la cola — el mismo mensaje reentregado para siempre, bloqueando de frente todo lo que viene detrás — o (b) se ackea/descarta igual solo para desbloquear la cola, perdiendo datos en silencio. Una DLQ da una tercera opción: ponerlo en cuarentena, mantener la cola principal fluyendo, y lidiar con el mensaje malo después.

**Cómo se cablea (ejemplo SQS, coincide con la `redrive policy` mencionada arriba):**

```typescript
// Queue definition (infra-as-code, e.g. CDK/Terraform, not runtime code)
const dlq = new sqs.Queue(this, "ClicksDLQ", {
  retentionPeriod: Duration.days(14), // DLQs typically keep messages longer than the main queue
});

const clicksQueue = new sqs.Queue(this, "ClicksQueue", {
  deadLetterQueue: {
    queue: dlq,
    maxReceiveCount: 5, // after 5 failed (unacked/timed-out) receives, redirect to dlq
  },
});
```

Nada en el código de aplicación tiene que detectar la falla y rutearla — SQS mismo cuenta cuántas veces se recibió un mensaje sin un `DeleteMessageCommand` correspondiente (es decir, expiró vía `VisibilityTimeout` o el consumidor lo nackeó explícitamente), y lo mueve a la DLQ al llegar a `maxReceiveCount`. El código consumidor queda igual que en el snippet de SQS anterior; simplemente nunca vuelve a ver ese mensaje después de la 5.ª falla.

Redis Streams no tiene primitiva de DLQ built-in — lo más parecido es código de aplicación que chequea `XPENDING`, y cuando el conteo de retries de una entrada excede un umbral, manualmente la `XADD`ea a un stream `clicks-dlq` separado y la `XACK`ea de la PEL original para que deje de reentregarse. Kafka: misma idea, usualmente implementada como convención (un topic `clicks-dlq`), no como feature del broker.

**Cuándo usar una:**

- La lógica del consumidor puede lanzar con input que parece válido pero está malformado (bugs de parseo/validación) y no querés que ese único mensaje malo trabe todo lo que viene atrás.
- Necesitás un rastro de auditoría de qué falló y por qué, en vez de descartes silenciosos — los mensajes de la DLQ típicamente se inspeccionan, loguean o re-procesan manualmente (o con un job automatizado) una vez arreglado el bug.
- Retry-con-backoff solo no alcanza porque la falla es determinística (dato malo), no transitoria (un blip) — reintentar infinitamente una falla determinística solo desperdicia tiempo del consumidor para siempre.
- No se necesita para analítica genuinamente fire-and-forget como el click tracking de este proyecto, donde perder un click malformado ocasional es aceptable — exactamente por eso la tabla de comparación de `Observabilidad` de arriba lista DLQ + métricas de lag como una preocupación de **producción**, no algo que este codebase implemente.

---

<a id="sub-7-9"></a>

### Árbol de decisión: ¿debería hacer await de esto?

```
¿El resultado se necesita para construir la respuesta?
  SÍ → await

¿La falla significa datos incorrectos (no solo datos faltantes)?
  SÍ → await

¿Es data financiera, de seguridad o de compliance?
  SÍ → await (y considerá una cola para durabilidad)

¿Podría esta operación ser lenta y llamarse a alta tasa?
  SÍ → await o usar cola acotada (p-limit / cola de mensajes)

¿El graceful shutdown necesita que esto complete?
  SÍ → await o usar un contador de drenado

¿Todo lo de arriba es NO?
  → fire-and-forget con .catch() + métricas
```

---

<a id="sub-7-10"></a>

### Este proyecto vs. analítica de producción

| Aspecto        | Este proyecto               | Producción                                  |
| -------------- | --------------------------- | ------------------------------------------- |
| Mecanismo      | Insert F&F a MongoDB        | Emitir a cola de mensajes (Kafka/SQS)       |
| Durabilidad    | Pierde lo en-vuelo en crash | La cola persiste; se reprocesa al reiniciar |
| Backpressure   | Ninguno — sin límite        | Profundidad de cola + tasa del consumidor   |
| Retry          | Ninguno — un solo intento   | El consumidor reintenta con backoff         |
| Observabilidad | Counter `clickWriteErrors`  | Dead-letter queue + métricas de lag         |
| Correctitud    | Aceptable: analítica        | Requerida: facturación, audit logs          |

El patrón F&F acá es correcto para un proyecto de aprendizaje / despliegue chico. A escala, reemplazarías el insert directo a MongoDB con un `redis.xadd()` a un Redis Stream (o topic de Kafka), y correrías un servicio consumidor de clicks separado.

---

**Referencias:**

- [MDN: Using Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_promises)
- [MDN: async/await](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises#async_and_await)
- [Node.js Event Loop — guía oficial](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick)
- [Node.js: the difference between process.nextTick() and setImmediate()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick#the-difference-between-setimmediate-and-settimeout)
- [Jake Archibald: Tasks, microtasks, queues and schedules](https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/)
- [libuv threadpool docs](https://docs.libuv.org/en/v1.x/threadpool.html)
- [Node.js: unhandledRejection](https://nodejs.org/api/process.html#event-unhandledrejection)
- [ESLint: no-floating-promises](https://typescript-eslint.io/rules/no-floating-promises/)
- [p-limit — control de concurrencia](https://github.com/sindresorhus/p-limit)
- [Redis Streams como cola de mensajes](https://redis.io/docs/data-types/streams/)
- [Libuv — la librería de I/O async debajo de Node](https://libuv.org/)
- [AWS SQS: Dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [AWS SQS: Visibility timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html)
- [Redis: XPENDING / XCLAIM (recuperación de consumer groups)](https://redis.io/commands/xpending/)

---

<a id="sec-8"></a>

## 8. Alertas y el Pipeline de Señales SRE

SRE = Site Reliability Engineering

<a id="sub-8-1"></a>

### Monitorear no es responder incidentes

Un dashboard lleno de métricas te dice qué anda mal **una vez que ya estás mirando**. La métrica que importa a las 3am es la que _despierta a alguien_. Sin alertas, tu **MTTD** (Mean Time To Detect) es "lo que tarde un humano en mirar Grafana de casualidad" — que en la práctica significa "hasta que se queje un cliente".

```
solo métricas:    ocurre incidente ──────────?─────────► alguien lo nota ──► fix
                                    (minutos a horas de falla silenciosa)

con alertas:      ocurre incidente ──► regla dispara ──► page ──► fix
                                    (segundos para detectar)
```

Este proyecto originalmente traía dashboards pero sin reglas de alerta — "visible en producción, no listo para producción". Las secciones de abajo describen el pipeline agregado para cerrar esa brecha.

<a id="sub-8-2"></a>

### El pipeline de alertas de Prometheus

```
┌────────────┐  evalúa      ┌──────────────────┐  dispara  ┌──────────────┐  rutea   ┌──────────┐
│ Prometheus │─────────────►│ reglas de alerta │──────────►│ Alertmanager │─────────►│ receiver │
│  (scrapes) │  cada 15s    │ (expr + for:)    │  HTTP push│ (dedup/group)│          │(Slack/PD)│
└────────────┘              └──────────────────┘           └──────────────┘          └──────────┘
```

- **Prometheus** evalúa el `expr` PromQL de cada regla cada `evaluation_interval` (15s acá).
- **Alertmanager** es un proceso _separado_. Deduplica (10 réplicas de la app disparando la misma alerta = una notificación), agrupa, silencia y rutea a los receivers. Prometheus solo decide _si_ una alerta dispara; Alertmanager decide _quién se entera y con qué frecuencia_.
- Un **receiver** es el destino de entrega (Slack, PagerDuty, Opsgenie, email, webhook).

<a id="sub-8-3"></a>

### `for:` — pending vs firing (debouncing)

Una regla con `for: 5m` no dispara en el instante en que su `expr` es verdadero. Primero pasa a **pending**, y solo transiciona a **firing** si la condición se mantiene verdadera los 5 minutos completos. Esto filtra blips transitorios — un solo scrape lento o un pico de CPU de 10 segundos no deberían pagear a nadie.

```
¿expr true? :  no  no  SÍ  SÍ  SÍ  SÍ  SÍ  no ...
estado      :  inactive──► pending(arranca timer)──► (se limpia antes de 5m → vuelve a inactive)

¿expr true? :  SÍ  SÍ  SÍ  SÍ  SÍ  SÍ (≥ 5m) ...
estado      :  pending ─────────────────────────► FIRING ──► Alertmanager
```

<a id="sub-8-4"></a>

### Alertas por síntoma vs por causa

| Estilo      | Alerta sobre…            | Ejemplo                           | Pro / Contra                                                                       |
| ----------- | ------------------------ | --------------------------------- | ---------------------------------------------------------------------------------- |
| **Síntoma** | Dolor visible al usuario | `HighErrorRate`, `HighLatencyP99` | Siempre accionable; pocos pages falsos. Preferido.                                 |
| **Causa**   | Una condición interna    | `RedisDown`, `TargetDown`         | Root-cause más rápido, pero puede pagear por cosas que los usuarios nunca sienten. |

Guía de Google SRE: **pagear por síntomas, diagnosticar con causas.** Demasiados pages por causa generan fatiga de alertas. Este proyecto mantiene las alertas por causa (`RedisDown`) en severidad más baja donde el cache degrada elegantemente (la DB sigue sirviendo), y trata `HighErrorRate` / `HighLatencyP99` como las señales reales.

<a id="sub-8-5"></a>

### Alertas por burn-rate de SLO (el siguiente nivel)

Un setup maduro no alerta por "tasa de error > 5% ahora mismo". Alerta por **qué tan rápido estás quemando tu presupuesto de error**. Si tu SLO es 99.9% de éxito (0.1% de presupuesto/mes), una alerta de burn-rate dispara cuando estás consumiendo ese presupuesto lo bastante rápido como para agotarlo antes de que termine la ventana — el burn rápido pagea inmediatamente, el burn lento abre un ticket. Esto evita tanto el flapping como las fugas lentas que pasan desapercibidas. (No implementado acá — `HighErrorRate` es un umbral simple — pero es la evolución natural.)

<a id="sub-8-6"></a>

### Mecánica — qué hace Prometheus en cada ciclo

Prometheus es un solo proceso corriendo un loop:

```
cada scrape_interval (15s):    GET http://app1:3000/metrics  → agrega samples a la TSDB local
cada evaluation_interval(15s): por cada regla: correr el expr PromQL contra la TSDB
                                 ├─ ¿el expr devuelve filas? → esos label-sets están "activos"
                                 ├─ activo < duración for:  → estado = PENDING (no se manda a ningún lado)
                                 └─ activo ≥ duración for:  → estado = FIRING  → push a Alertmanager
```

Puntos clave:

- Una alerta es **por fila de resultado**, no por regla. Si el expr devuelve 3 series, obtenés 3 instancias de alerta con labels distintos.
- Prometheus pushea las alertas al `/api/v2/alerts` de Alertmanager **y las sigue re-enviando en cada evaluación** mientras están firing (así un Alertmanager reiniciado re-aprende el estado). Cuando el expr deja de devolver la fila, Prometheus manda un `resolved`.
- **Solo FIRING se envía.** `pending` vive enteramente dentro de Prometheus — por eso una alerta pending nunca aparece en la UI de Alertmanager.

<a id="sub-8-7"></a>

### Mecánica — qué hace Alertmanager con una alerta firing

Alertmanager es un proceso _separado_ cuyo único trabajo es convertir un stream de alertas firing en la cantidad correcta de notificaciones útiles:

```
alerta firing entra ─► [ árbol de rutas ] ─► [ group_by ] ─► [ wait/dedup/inhibit/silence ] ─► receiver
```

- **Routing** — un árbol matchea labels de alerta con un receiver (ej. `severity=critical` → PagerDuty, todo lo demás → Slack).
- **Grouping** (`group_by`) — 10 réplicas de la app disparando `RedisDown` colapsan en **una** notificación, no diez. `group_wait` (10s) retiene la primera notificación brevemente para que las alertas relacionadas se agrupen; `group_interval` controla los follow-ups; `repeat_interval` (1h acá) es cada cuánto re-notifica si sigue firing.
- **Dedup** — alertas idénticas de múltiples Prometheus (pares HA) se vuelven una.
- **Silences** — un humano mutea alertas que matcheen durante una ventana (deploys, mantenimiento).
- **Inhibition** — una alerta de más alto nivel suprime ruido (ej. `TargetDown` inhibe `HighLatencyP99` para el mismo target — no tiene sentido pagear por latencia cuando está caído).

En este proyecto el receiver es `null` (sin paging real), así que las alertas firing aparecen en la **UI** de Alertmanager pero no van a ningún otro lado — suficiente para probar el pipeline.

<a id="sub-8-8"></a>

### Trampa — `for:` debe ser más corto que la vida del dato en la ventana de rate

Esto hace tropezar a todos (y es la razón por la que una ráfaga rápida de `npm run gen:errors` no dispara `HighErrorRate`). La regla es:

```
HighErrorRate:  rate(http_requests_total{5xx}[5m]) / rate(...[5m]) > 0.05   for: 5m
```

Una ráfaga de errores de 5 segundos cae en la ventana `[5m]`, así que el ratio pica y la alerta pasa a **pending**. Pero los samples de la ráfaga solo _permanecen_ en una ventana `[5m]` durante 5 minutos — envejecen y salen exactamente cuando el timer de `for: 5m` completa, el expr pasa a false, y la alerta se resuelve **sin haber disparado nunca**.

```
t=0    50 errores disparados (ráfaga de 5s), Mongo despausado
t=0    expr true  → PENDING (arranca el timer de for:)
t=5m   el timer de for: completaría... pero los 50 errores acaban de salir de la ventana rate[5m]
       expr → false → vuelve a inactive. NUNCA DISPARA.
```

**Regla práctica:** para disparar una alerta con `for: D` y `rate[W]`, la condición debe mantenerse **al menos D**, lo que significa que los eventos subyacentes deben seguir ocurriendo durante ≳ D (no solo W). Para disparar `HighErrorRate` de verdad: sostener errores > 5 min (`node scripts/gen-errors.js 2000 5` ≈ 6.7 min de Mongo pausado), o bajar el `for:`. En contraste, `TargetDown`/`RedisDown` disparan confiablemente en `test-alerts.sh` porque su condición (`up==0`, `redis_circuit_open==1`) es _level-triggered_ — se mantiene verdadera todo el tiempo que la dependencia esté caída, no es una tasa que decae.

<a id="sub-8-9"></a>

### Cómo lo hace este proyecto (Alertas)

`observability/prometheus.rules.yml` define seis alertas; `observability/prometheus.yml` cablea `rule_files` + un target de `alertmanagers`; `observability/alertmanager.yml` define un receiver `null` no-op (el challenge no tiene integración de paging real — demuestra el pipeline completo). El servicio `alertmanager` corre en `docker-compose.yml` (loopback `:9093`).

| Alerta                     | Tipo    | Expr (esencia)                       | `for:` |
| -------------------------- | ------- | ------------------------------------ | ------ |
| `TargetDown`               | causa   | `up{job="url-shortener"} == 0`       | 1m     |
| `HighErrorRate`            | síntoma | ratio de 5xx > 5%                    | 5m     |
| `RedisDown`                | causa   | `redis_circuit_open == 1`            | 2m     |
| `RedirectCacheHitRatioLow` | causa   | hit ratio de url < 80%               | 10m    |
| `ClickWriteErrors`         | causa   | `rate(click_write_errors_total) > 0` | 5m     |
| `HighLatencyP99`           | síntoma | p99 > 0.5s                           | 5m     |

`scripts/test-alerts.sh` prueba el pipeline end-to-end: para `app1` / `redis`, pollea `/api/v1/alerts` de Prometheus hasta que la alerta esté `firing`, confirma que propagó a `/api/v2/alerts` de Alertmanager, después restaura y confirma que se limpia.

**Referencias:**

- [Prometheus: Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Prometheus: Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Google SRE Workbook: Alerting on SLOs (burn rate)](https://sre.google/workbook/alerting-on-slos/)
- [My Philosophy on Alerting — Rob Ewaschuk](https://docs.google.com/document/d/199PqyG3UsyXlwieHaqbGiWVa8eMWi8zzAn0YfcApr8Q/)

---

<a id="sec-9"></a>

## 9. RED y USE — Dos Métodos para Elegir Métricas

No podés graficar todo. Dos modelos mentales complementarios te dicen _qué_ señales importan.

<a id="sub-9-1"></a>

### RED — para servicios orientados a requests (la app)

| Letra        | Métrica                       | Este proyecto                                           |
| ------------ | ----------------------------- | ------------------------------------------------------- |
| **R**ate     | Requests por segundo          | `rate(http_requests_total[1m])` (paneles read vs write) |
| **E**rrors   | Requests fallidos por segundo | `rate(http_requests_total{status=~"5.."}[1m])`          |
| **D**uration | Distribución de latencia      | `histogram_quantile(…, http_duration_seconds_bucket)`   |

RED responde: _"¿Mis usuarios están recibiendo respuestas rápidas y correctas?"_ Está orientado a síntomas — las mismas tres señales sobre las que disparan las mejores alertas (§8).

<a id="sub-9-2"></a>

### USE — para recursos (CPU, memoria, disco, pools, el event loop)

| Letra           | Significado                                                | Este proyecto                                                         |
| --------------- | ---------------------------------------------------------- | --------------------------------------------------------------------- |
| **U**tilization | % del tiempo que el recurso está ocupado                   | CPU vía `collectDefaultMetrics`, límite `cpus` del contenedor         |
| **S**aturation  | Trabajo encolado/esperando que el recurso no puede atender | **lag del event loop** (`nodejs_eventloop_lag_*`), RSS vs `mem_limit` |
| **E**rrors      | Eventos de error del recurso                               | `redis_errors_total`, `click_write_errors_total`                      |

USE responde: _"¿Algún recurso es el cuello de botella?"_

<a id="sub-9-3"></a>

### Por qué el lag del event loop es _la_ señal de saturación en Node

Node es single-threaded. Si un handler sincrónico acapara la CPU, el event loop no puede atender los callbacks de I/O pendientes — se encolan. El **lag del event loop** mide exactamente esa demora: la brecha entre cuándo un timer _debería_ disparar y cuándo _realmente_ dispara. Lag creciente significa que el proceso está saturado aunque el CPU% se vea moderado. Es el clásico page de Node a las 3am, por eso se agregó un panel para `nodejs_eventloop_lag_p99_seconds` (viene gratis con `collectDefaultMetrics`, solo que no estaba graficado).

<a id="sub-9-4"></a>

### Cómo lo hace este proyecto (RED/USE)

`src/observability/metrics.ts` registra las métricas RED explícitamente; `collectDefaultMetrics` provee las señales USE (lag del event loop, RSS, GC, CPU). El dashboard de Grafana (`observability/grafana/dashboards/url-shortener.json`) ahora tiene paneles para lag p99 del event loop y RSS del proceso junto a los paneles RED existentes.

**Referencias:**

- [The RED Method — Tom Wilkie (Grafana/Weave)](https://www.weave.works/blog/the-red-method-key-metrics-for-microservices-architecture/)
- [The USE Method — Brendan Gregg](https://www.brendangregg.com/usemethod.html)
- [Google SRE Book: The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/#xref_monitoring_golden-signals)
- [Node.js perf_hooks: monitorEventLoopDelay](https://nodejs.org/api/perf_hooks.html#perf_hooksmonitoreventloopdelayoptions)

---

<a id="sec-10"></a>

## 10. Liveness vs Readiness Probes

Estos dos health checks responden **preguntas distintas**, y confundirlos causa outages.

| Probe         | Pregunta                                            | Si falla, el orquestador…                    | ¿Chequea dependencias? |
| ------------- | --------------------------------------------------- | -------------------------------------------- | ---------------------- |
| **Liveness**  | "¿Este proceso está vivo / no deadlockeado?"        | **Reinicia el contenedor**                   | **No**                 |
| **Readiness** | "¿Esta instancia puede servir tráfico ahora mismo?" | **La saca del load balancer** (sin reinicio) | **Sí**                 |

<a id="sub-10-1"></a>

### La trampa de la tormenta de reinicios

Suponé que usás **un solo** endpoint `/health` que pingea MongoDB, y lo cableás al liveness check del contenedor. Mongo tiene un blip de 30 segundos:

```
Mongo tiene un blip
  → /health devuelve 503 en cada instancia de la app
  → el liveness probe falla en todos lados
  → el orquestador MATA y reinicia todos los contenedores de la app simultáneamente
  → caches fríos, tormentas de reconexión, trabajo en vuelo perdido
  → el blip se convirtió en un outage completo
```

La app estaba **bien** — solo su dependencia tuvo un hipo. Reiniciarla no arregló nada y empeoró todo. El liveness no debe depender de **nada externo**. El readiness es donde van los chequeos de dependencias: un 503 ahí solo deja de rutear tráfico nuevo a esa instancia hasta que Mongo se recupere — sin reinicio, sin tormenta.

<a id="sub-10-2"></a>

### Cómo lo hace este proyecto (Probes)

`src/server.ts`:

- `GET /livez` → devuelve `{status:'ok'}` **sin chequeos de dependencias**. El healthcheck de Docker en `docker-compose.yml` apunta acá, así que un blip de Mongo nunca reinicia una app sana.
- `GET /health` → pingea Mongo, devuelve 503 si es inalcanzable. Este es el probe de **readiness** — pensado para que el load balancer (NGINX) decida el ruteo.

```
docker-compose healthcheck:  curl -f http://localhost:3000/livez   ← decisión de reinicio (sin deps)
ruteo del load balancer:     GET /health                            ← decisión de tráfico (con deps)
```

**Mapeo a Kubernetes:** `livenessProbe` → `/livez`, `readinessProbe` → `/health`. (Un tercero, `startupProbe`, protege a apps de arranque lento para que el liveness no las mate durante el boot — el `start_period: 15s` de compose es el equivalente acá.)

**Referencias:**

- [Kubernetes: Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Google SRE: Health checking](https://sre.google/sre-book/load-balancing-datacenter/)
- [Liveness probes are dangerous — Colin Breck](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/)

---

<a id="sec-11"></a>

## 11. Cardinalidad de Labels de Métricas

**Cardinalidad** = la cantidad de series temporales distintas que produce una métrica. Prometheus guarda **una serie temporal por cada combinación única de valores de labels**, en memoria. Esta es la forma más fácil de tirar abajo Prometheus.

```
http_requests_total{method, route, status}

methods: ~4   ×   routes: ~5   ×   statuses: ~10   =   ~200 series   ✅ bien
```

Ahora imaginá que `route` contuviera el **path crudo de la URL** en vez de un _patrón_ de ruta:

```
http_requests_total{route="/aB3xK"}      ← cada short code es una serie nueva
http_requests_total{route="/9zQ1p"}
http_requests_total{route="/scan-attempt-47281"}   ← cada probe de scanner también
...millones de series → Prometheus hace OOM
```

Esto es una **bomba de cardinalidad**. Un bot escaneando paths aleatorios podría acuñar series sin límite y crashear tu monitoreo — convirtiendo un ataque a tu app en un ataque a tu observabilidad.

<a id="sub-11-1"></a>

### Las reglas

1. **Los labels deben estar acotados.** Nunca pongas URLs crudas, IDs de usuario, emails, request IDs, timestamps o mensajes de error completos en un label.
2. **Usá patrones, no valores.** `route="/:shortUrl"` (una serie) no `route="/aB3xK"` (∞).
3. **Bucketeá dimensiones de alta cardinalidad.** `status_class="2xx"` (4 valores) en vez de, o junto a, el `status` crudo solo donde el valor crudo se necesite genuinamente.
4. **Los datos de alta cardinalidad van en logs/trazas,** no en métricas. (Ver §12 — para eso está el `X-Request-Id` en Loki.)

<a id="sub-11-2"></a>

### Cómo lo hace este proyecto (Cardinalidad)

`src/middleware/metrics.ts`:

- `routeLabel()` devuelve `req.route?.path` (siempre un **patrón** como `/:shortUrl`) y colapsa el catch-all sin match a un único label fijo `'unmatched'` — así el tráfico de scanners golpeando paths aleatorios nunca puede expandir la cardinalidad.
- `statusClass()` mapea el status numérico a `2xx/3xx/4xx/5xx` para el label del **histograma de duración**, manteniendo la latencia divisible-por-resultado sin una dimensión de status sin límite. El `status` crudo queda solo en el _counter_ (donde ~10 códigos es aceptable).

`src/middleware/metrics.test.ts` verifica la guarda: dispara tráfico con match + de scanner y comprueba que los segmentos de path crudos **nunca** aparecen como valores de label.

**Referencias:**

- [Prometheus: Naming and labels best practices](https://prometheus.io/docs/practices/naming/)
- [Prometheus: Cardinality is key](https://www.robustperception.io/cardinality-is-key/)
- [Grafana: Avoiding high cardinality](https://grafana.com/blog/2022/02/15/what-are-cardinality-spikes-and-why-do-they-matter/)

---

<a id="sec-12"></a>

## 12. Correlación de Logs, Métricas y Trazas

Los "tres pilares de la observabilidad" responden preguntas distintas:

| Pilar        | Responde                                         | Cardinalidad           | Este proyecto                        |
| ------------ | ------------------------------------------------ | ---------------------- | ------------------------------------ |
| **Métricas** | "¿Algo anda mal, y cuánto?"                      | Baja (labels acotados) | Prometheus                           |
| **Logs**     | "¿Qué pasó exactamente en este request?"         | Alta (formato libre)   | pino → Loki                          |
| **Trazas**   | "¿Dónde se fue el tiempo a través de servicios?" | Alta (por span)        | (no implementado — un solo servicio) |

Una métrica te dice que el p99 de latencia picó. No puede decirte _qué_ requests fueron lentos. Una línea de log tiene el detalle pero necesitás una forma de **pivotear** del pico a las líneas. Ese puente es un **request ID** compartido.

```
el cliente reporta "el redirect estuvo lento a las 14:32"
        │
        ▼
la respuesta llevaba  X-Request-Id: 7f3a…        ← el puente
        │
        ▼
Loki:  {app="app1"} |= "7f3a…"                   ← las líneas de log exactas de ese request
```

Para un **servicio único**, un request ID end-to-end es el 80% barato del tracing distribuido. El tracing completo (OpenTelemetry, propagando un contexto de traza a través de saltos entre servicios) importa cuando hacés fan-out a múltiples servicios — es el próximo paso natural, no necesario acá.

<a id="sub-12-1"></a>

### Cómo lo hace este proyecto (Correlación)

- `src/middleware/logger.ts` — el `genReqId` de `pino-http` asigna un UUID `req.id` a cada request; cada línea de log de ese request lo lleva.
- `src/server.ts` — un middleware setea `res.setHeader('X-Request-Id', req.id)` para que el id sea visible a los clientes y en trazas de red. Corre **después** del logger (que crea el id) y antes de los route handlers.
- El id fluye a Loki vía Promtail, así que un request reportado por un cliente es greppeable end-to-end.

**Referencias:**

- [Three pillars of observability — Honeycomb](https://www.honeycomb.io/blog/observability-101-terminology-and-concepts)
- [OpenTelemetry: Context propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [Grafana Loki: LogQL](https://grafana.com/docs/loki/latest/query/)
- [pino-http: request id](https://github.com/pinojs/pino-http#pinohttpopts-stream)

---

<a id="sec-13"></a>

## 13. Hardening de Contenedores — Radio de Impacto y Aislamiento

La mayoría de estos son cambios de una línea que convierten un incidente chico en uno contenido en vez de uno a nivel host.

<a id="sub-13-1"></a>

### Contenedores non-root

Por defecto el proceso de un contenedor corre como **root dentro del contenedor**. Si un atacante logra RCE (o una dependencia está comprometida), root-en-contenedor es un radio de impacto mucho mayor — facilita exploits de escape de contenedor y le permite al proceso alterar archivos montados. Correr como usuario sin privilegios es defensa en profundidad.

```dockerfile
RUN chown -R node:node /app
USER node          # node:alpine ships this user; the app never needs root
```

<a id="sub-13-2"></a>

### Límites de recursos — el problema del vecino ruidoso / OOM

Los contenedores en un host comparten el kernel y la RAM física. **Sin límites**, una fuga de memoria (o Redis llenándose hasta su `--maxmemory`) puede consumir toda la RAM del host y el OOM killer empieza a matar _otros_ contenedores — tu DB muere porque tu app tuvo una fuga.

```yaml
mem_limit: 512m # app capped — a leak kills only the app, host stays up
cpus: 1.0
# redis mem_limit set ABOVE its --maxmemory 2gb so it evicts (LFU, §5) before OOM-kill
```

Los límites convierten "falla ilimitada del host" en "un contenedor se reinicia".

<a id="sub-13-3"></a>

### Comparación de secretos timing-safe

Un `token !== expected` ingenuo retorna apenas encuentra el primer byte distinto. El **tiempo que tarda filtra cuántos bytes iniciales coincidieron** — un atacante puede recuperar un secreto byte por byte (un side-channel de timing). Usá una comparación de tiempo constante:

```ts
function tokensMatch(provided: string, expected: string): boolean {
  const a = Buffer.from(provided),
    b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b); // always compares all bytes
}
```

(La longitud se compara primero porque `timingSafeEqual` _requiere_ buffers de igual longitud; la longitud en sí es de bajo valor como filtración.)

<a id="sub-13-4"></a>

### Binding de puertos solo a loopback

`ports: ["27017:27017"]` bindea a `0.0.0.0` — el datastore es alcanzable desde **cualquier** interfaz de red, incluidas las públicas, a menudo sin auth. Bindear a `127.0.0.1` mantiene el puerto disponible para debugging local (`mongosh`, `redis-cli`) mientras lo hace inalcanzable desde la red. El tráfico entre contenedores no se ve afectado — usa el DNS interno de Docker, no el puerto publicado del host.

```yaml
ports:
  - "127.0.0.1:27017:27017" # localhost only, not the world
```

<a id="sub-13-5"></a>

### Cómo lo hace este proyecto (Hardening)

- `Dockerfile`: `chown` + `USER node` en el stage de runtime (non-root).
- `docker-compose.yml`: `mem_limit`/`cpus` en cada servicio; Mongo, Redis, Prometheus, Alertmanager y los exporters bindeados a `127.0.0.1` (solo NGINX `:80` y Grafana `:3001` miran al mundo); la contraseña admin de Grafana ya no defaultea a `admin`.
- `src/server.ts`: `tokensMatch()` usa `crypto.timingSafeEqual` para el token de `/metrics`.

**Referencias:**

- [Docker: Run containers as a non-root user](https://docs.docker.com/build/building/best-practices/#user)
- [Docker Compose: resource constraints](https://docs.docker.com/compose/compose-file/compose-file-v3/#resources)
- [Node.js crypto.timingSafeEqual](https://nodejs.org/api/crypto.html#cryptotimingsafeequala-b)
- [OWASP: Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)

---

<a id="sec-14"></a>

## 14. Flujo de Datos de Métricas — Modelo Pull, Fuente vs Vista

Una confusión común: _"si Grafana ya muestra los datos, ¿el endpoint `/metrics` de la app no es un bypass sin sentido?"_ No — es la **fuente**. La dependencia corre en la otra dirección.

```
┌──────────────┐  scrape cada 15s    ┌──────────────┐   query PromQL   ┌──────────┐
│ app /metrics │────────────────────►│  Prometheus  │─────────────────►│ Grafana  │
│   FUENTE     │   (HTTP GET, pull)  │ ALMACENAMIENTO│  (al cargar el   │  VISTA   │
│ snapshot vivo│                     │ 30d historia │   dashboard)     │ (dibuja) │
└──────────────┘                     └──────────────┘                  └──────────┘
```

| Capa           | Rol                                                  | ¿Tiene historia?       | ¿Genera datos?     |
| -------------- | ---------------------------------------------------- | ---------------------- | ------------------ |
| app `/metrics` | **produce** los números (counters/gauges in-process) | No — instantáneo       | Sí (el origen)     |
| Prometheus     | **scrapea y almacena**                               | Sí (TSDB)              | No — solo registra |
| Grafana        | **consulta y dibuja**                                | No (lee de Prometheus) | **No**             |

Grafana no renderiza _nada propio_. Matá `/metrics` → Prometheus scrapea vacío → Grafana queda en blanco. "Grafana muestra los mismos datos" es todo el punto: son _esos_ datos, almacenados y graficados. Golpear `/metrics` directo es solo para debugging ("¿la app siquiera está exponiendo el counter X?") sin el lag de scrape→almacenar→renderizar.

<a id="sub-14-1"></a>

### Pull vs push

Prometheus **pullea** (scrapea un endpoint HTTP) en vez de que la app **pushee**. Beneficios: la app queda tonta (solo expone valores actuales, sin egreso de red hacia un backend de métricas); las fallas de scrape son en sí una señal (`up == 0` → la alerta `TargetDown`); y cualquier herramienta puede leer `/metrics` independientemente. El tradeoff — los jobs de vida corta que mueren entre scrapes necesitan un Pushgateway — no aplica a un web server de larga vida.

<a id="sub-14-2"></a>

### Dos familias en `/metrics`

| Familia                          | Ejemplos                                                                                                              | Método                                      | Por qué                                          |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------------------ |
| **Métricas de app** (explícitas) | `http_requests_total`, `cache_hits_total`, `redis_circuit_open`                                                       | definidas en `src/observability/metrics.ts` | RED + señales de dominio                         |
| **Métricas default** (auto)      | `process_resident_memory_bytes`, `nodejs_eventloop_lag_p99_seconds`, `nodejs_gc_duration_seconds`, `process_open_fds` | `collectDefaultMetrics()`                   | USE/saturación — _por qué_ el proceso está lento |

El bloque `process_*` / `nodejs_*` **no es ruido** — es la vista de recursos (USE, §9) que las métricas de app no pueden proveer: lag del event loop (loop bloqueado), RSS (fuga), pausas de GC (latencia), cantidad de fds (fuga de conexiones). Los paneles de lag del event loop y RSS del dashboard leen exactamente estas.

<a id="sub-14-3"></a>

### Cómo lo hace este proyecto (Métricas)

`src/observability/metrics.ts` construye un registry de `prom-client` (`collectDefaultMetrics` para la familia USE + métricas RED/de dominio explícitas). `src/server.ts` lo sirve en `/metrics` detrás de un bearer token. `observability/prometheus.yml` scrapea `app1:3000` cada 15s; `observability/grafana/provisioning/datasources` apunta Grafana a Prometheus.

**Referencias:**

- [Prometheus: Overview & data model (pull)](https://prometheus.io/docs/introduction/overview/)
- [Prometheus: Why pull over push](https://prometheus.io/docs/introduction/faq/#why-do-you-pull-rather-than-push)
- [prom-client: default metrics](https://github.com/siimon/prom-client#default-metrics)
- [Grafana: Prometheus data source](https://grafana.com/docs/grafana/latest/datasources/prometheus/)

---

<a id="sec-15"></a>

## 15. Parseo de URLs WHATWG y Normalización de Entradas

<a id="sub-15-1"></a>

### Qué es el estándar WHATWG URL

El **WHATWG URL Standard** (`https://url.spec.whatwg.org/`) es la especificación viviente que siguen los navegadores y Node.js al parsear URLs. Reemplazó el enfoque más viejo de RFC 3986 para la plataforma web y es lo que usa `new URL(string)` en JavaScript.

Objetivo de diseño clave: **ser permisivo en lo que aceptás, pero producir una salida canónica**. Esto significa que el parser arregla silenciosamente muchos inputs en vez de rechazarlos:

| Input                            | `new URL(input).href`         | Qué pasó                               |
| -------------------------------- | ----------------------------- | -------------------------------------- |
| `"https://EXAMPLE.COM/Path"`     | `"https://example.com/Path"`  | host pasado a minúsculas               |
| `"https://example.com/a%20b"`    | `"https://example.com/a%20b"` | percent-encoding preservado            |
| `"https://example.com/a b"`      | `"https://example.com/a%20b"` | espacio en el path codificado          |
| `"  https://example.com  "`      | `"https://example.com/"`      | **whitespace inicial/final eliminado** |
| `"https://example.com/./a/../b"` | `"https://example.com/b"`     | path normalizado                       |

Las últimas dos filas son las críticas. El parser elimina el whitespace circundante y resuelve los dot-segments antes incluso de empezar a interpretar los componentes de la URL.

<a id="sub-15-2"></a>

### Zod `.url()` valida pero no normaliza

El validador `.url()` de Zod llama `new URL(input)` internamente — pero solo para **chequear** si el parseo tiene éxito. Devuelve el **string original** sin cambios:

```ts
z.string().url().parse("  https://example.com  ");
// returns: "  https://example.com  "   ← original, with spaces
```

Este es el comportamiento correcto para un validador: el trabajo de Zod es decir sí/no, no reescribir tus datos — salvo que se lo digas con un transform.

<a id="sub-15-3"></a>

### El bug de dedup que esto causa

Este proyecto deduplica hasheando `longUrl` con SHA-256:

```
"https://example.com" → SHA-256 → "abc123..." → guardado como índice longUrlHash
```

Si se hashea el string crudo (sin trim):

```
"https://example.com"   → SHA-256 → "abc123..." → shortCode: "xK3p"
"https://example.com "  → SHA-256 → "def456..." → shortCode: "zQ9r"   ← DISTINTO
```

Dos short codes distintos ahora redirigen al **mismo destino** — un miss de dedup. El parser WHATWG trata las dos URLs como idénticas (el whitespace se elimina antes de parsear), pero el hash SHA-256 ve dos secuencias de bytes distintas.

<a id="sub-15-4"></a>

### El fix: `.trim()` antes de `.url()`

```ts
export const longUrlSchema = z
  .string()
  .trim()   // ← normalize BEFORE validation
  .url()
  .max(2048)
  ...
```

`.trim()` es un transform de Zod — reescribe el valor, así que `.url()` (y todo lo que sigue) ve el string limpio. Ahora tanto `"https://example.com"` como `"https://example.com "` hashean al mismo valor y pegan en el índice de dedup.

<a id="sub-15-5"></a>

### La regla general: normalizar entradas en el borde

```
           ANTES de normalizar: "https://x.com/hello "
                                           │
        ┌──────────────────────────────────▼───────┐
        │         Borde de validación              │
        │  1. trim()  → "https://x.com/hello"     │  ← forma canónica
        │  2. url()   → pasa                      │
        │  3. max()   → pasa                      │
        └──────────────────────────────────────────┘
                                           │
                  guardado / hasheado: "https://x.com/hello"  ✅
```

La normalización silenciosa del parser WHATWG es útil en un navegador (UX indulgente), pero en un server que guarda un hash del input crea un desajuste entre "lo que ve el parser" y "lo que guardás". Siempre normalizá a la forma canónica **antes** de hashear o guardar.

<a id="sub-15-6"></a>

### Cuándo ir más lejos

`.trim()` maneja el whitespace. Para canonicalización completa de URLs (remoción de fragmentos, esquema a minúsculas, normalización de trailing-slash, orden del query string), podrías hacer:

```ts
.transform(u => new URL(u).href)   // full WHATWG normalization
```

Este proyecto se queda en `.trim()` — alcanza para el caso de dedup y evita reescribir URLs de formas inesperadas (ej. normalizar `%2F` en paths).

<a id="sub-15-7"></a>

### Cómo lo hace este proyecto (Normalización)

`src/utils/validators.ts` — `.trim()` agregado antes de `.url()` en `longUrlSchema`.
`src/utils/validators.test.ts` — el test verifica que una URL con whitespace circundante produce el mismo valor `.data` que la URL limpia, confirmando consistencia de dedup.

**Referencias:**

- [WHATWG URL Standard](https://url.spec.whatwg.org/)
- [MDN: URL() constructor](https://developer.mozilla.org/en-US/docs/Web/API/URL/URL)
- [Zod: string validations & transforms](https://zod.dev/?id=strings)
- [Node.js: URL module (implementación WHATWG)](https://nodejs.org/api/url.html#the-whatwg-url-api)

---

<a id="sec-16"></a>

## 16. Diseño de Histogramas de Prometheus — Buckets, Labels y Cobertura de Cola

<a id="sub-16-1"></a>

### Por qué histogramas para latencia (no gauges, no counters)

Tres tipos de métrica pueden registrar una duración:

| Tipo          | Qué guarda                         | ¿Puede calcular percentiles? | Memoria por métrica |
| ------------- | ---------------------------------- | ---------------------------- | ------------------- |
| **Gauge**     | un valor actual                    | No                           | 1 serie             |
| **Counter**   | total acumulado                    | No                           | 1 serie             |
| **Histogram** | conteo de observaciones por bucket | Sí (aproximado)              | 1 serie por bucket  |
| **Summary**   | cuantiles precalculados in-process | Sí (exacto, ventana fija)    | 1 serie por cuantil |

Los histogramas ganan para latencia porque:

- Los percentiles se computan **server-side** en PromQL — podés calcularlos a través de instancias y ventanas de tiempo arbitrarias después del hecho.
- Los summaries calculan cuantiles en el **proceso de la app** — no se pueden agregar entre réplicas y la ventana del cuantil se fija al instrumentar.
- El tradeoff: los percentiles de histograma son **aproximados** (interpolados dentro de un bucket). La precisión mejora con buckets más finos alrededor de los valores que te importan.

<a id="sub-16-2"></a>

### Cómo funcionan los buckets

Un histograma crea un counter por límite de bucket. Cada counter trackea cuántas observaciones cayeron **en o debajo** de ese límite (acumulativo):

```
observe(0.003s):
  http_duration_seconds_bucket{le="0.005"} += 1
  http_duration_seconds_bucket{le="0.01"}  += 1
  http_duration_seconds_bucket{le="0.025"} += 1
  ... all larger buckets also increment (cumulative)

observe(0.150s):
  http_duration_seconds_bucket{le="0.25"} += 1
  http_duration_seconds_bucket{le="0.5"}  += 1
  ... larger buckets only
```

`histogram_quantile(0.99, rate(http_duration_seconds_bucket[5m]))` interpola el valor p99 desde estos conteos acumulativos. Si el 99% de las observaciones caen en el bucket `(0.1, 0.25]`, Prometheus asume distribución uniforme dentro de ese rango e interpola. **La precisión se degrada si el bucket es demasiado ancho.**

<a id="sub-16-3"></a>

### Diseño de buckets: cubrir tu SLO, extender la cola

Los buckets default de `prom-client` paran en **10 segundos**. Para un URL shortener con objetivo de p99 sub-10ms, el rango interesante es `[0.005, 0.5]`. Pero igual necesitás buckets de cola — no para el tráfico normal, sino porque:

1. **Las alertas sobre p99** necesitan un bucket finito donde interpolar. Sin un bucket `10`, los requests que tardan 7 segundos se apilarían todos en `+Inf` y la expresión PromQL de tu alerta HighLatencyP99 no puede distinguir "un poco lento" de "completamente trabado".
2. **Investigación de incidentes**: cuando algo anda mal, querés saber _qué tan_ mal — "el p99 es 3s" vs "el p99 es 8s" lleva a diagnósticos distintos.

```
Buckets default de prom-client: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1]
                                                                           ↑
                                                        para acá — sin cola

Buckets de este proyecto:       [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
                                                                            └───────────┘
                                                                          cobertura de cola
```

La regla: **tu umbral de SLO más lento debe quedar dentro de un bucket**, no pasado el final.

<a id="sub-16-4"></a>

### El label `status_class` — segmentación por resultado sin explosión de cardinalidad

Agregar `status` (código HTTP crudo) como label de un histograma es tentador — te gustaría tener percentiles de latencia separados para requests exitosos vs fallidos. El problema:

```
Sin status_class:
  http_duration_seconds_bucket{method, route, le}
  ~4 methods × ~5 routes × 11 buckets = 220 series ✅

Con status crudo:
  http_duration_seconds_bucket{method, route, status, le}
  ~4 × ~5 × ~10 statuses × 11 buckets = 2200 series — 10× más
  + los scanners pueden probar paths de status inusuales → sin límite
```

`status_class` colapsa todos los códigos de status a cuatro valores (`2xx`, `3xx`, `4xx`, `5xx`):

```
http_duration_seconds_bucket{method, route, status_class, le}
~4 × ~5 × 4 × 11 = 880 series — acotado y útil
```

Ahora podés escribir:

```promql
histogram_quantile(0.99,
  sum by (le) (
    rate(http_duration_seconds_bucket{status_class="2xx"}[5m])
  )
)
```

— latencia p99 **solo de requests exitosos**, filtrando los paths de error que siempre son lentos (ej. paths con la base caída) para que no inflen la métrica del SLO.

<a id="sub-16-5"></a>

### Cómo `status_class` habilita la alerta HighErrorRate

La alerta `HighErrorRate` usa `http_requests_total` (el counter, que mantiene el `status` crudo). El `status_class` del histograma es aparte — está para el **panel de SLO** de latencia, no para la alerta de tasa de error. Las dos métricas sirven propósitos distintos:

| Métrica                 | Label usado    | Propósito                                                |
| ----------------------- | -------------- | -------------------------------------------------------- |
| `http_requests_total`   | `status` crudo | alerta de tasa de error — `rate(...{status=~"5.."}[5m])` |
| `http_duration_seconds` | `status_class` | SLO de latencia — `histogram_quantile(0.99, ...)`        |

<a id="sub-16-6"></a>

### Decisiones de diseño de paneles en Grafana

Tres paneles nuevos agregados en esta sesión:

| Panel                     | Métrica                            | Por qué                                                                  |
| ------------------------- | ---------------------------------- | ------------------------------------------------------------------------ |
| Estado del circuito Redis | `redis_circuit_open`               | gauge binario: 1=roto, 0=ok — degradación visible al instante            |
| Lag p99 del event loop    | `nodejs_eventloop_lag_p99_seconds` | señal de saturación — loop bloqueado → respuesta lenta aun con DB rápida |
| RSS del proceso           | `process_resident_memory_bytes`    | detección de fugas — RSS creciendo por horas = fuga de memoria           |

El filtro del panel de logs de error en Loki se cambió de `{level="50"}` (solo error) a `{level=~"40|50"}` (warn + error). pino emite niveles numéricos: `30`=info, `40`=warn, `50`=error. Las fallas de escritura de clicks loguean en `warn` (40) — serían invisibles con el filtro viejo.

<a id="sub-16-7"></a>

### Cómo lo hace este proyecto (Histogramas)

`src/observability/metrics.ts` — el Histogram `httpDuration` definido con el array de buckets extendido y `status_class` en `labelNames`.
`src/middleware/metrics.ts` — el helper `statusClass(code)` mapea `Math.floor(code/100)` al string `"2xx"`. `routeLabel(req)` devuelve `req.route?.path ?? 'unmatched'`.
`observability/grafana/dashboards/url-shortener.json` — paneles 11–13 para estado del circuito, lag del event loop y RSS; filtro del panel 10 actualizado a warn+error.

**Referencias:**

- [Prometheus: Histogram vs Summary](https://prometheus.io/docs/practices/histograms/)
- [Prometheus: histogram_quantile](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)
- [prom-client: Histogram](https://github.com/siimon/prom-client#histogram)
- [Google SRE Book: Latency SLOs](https://sre.google/sre-book/service-level-objectives/)
- [Robust Perception: Cardinality is key](https://www.robustperception.io/cardinality-is-key/)

---

<a id="sec-17"></a>

## 17. Monolito vs Microservicios vs Arquitectura Orientada a Eventos

Estos son tres ejes separados, no tres cajas mutuamente excluyentes:

- **Monolito vs Microservicios** = cómo dibujás los límites de _despliegue_ (un proceso vs muchos).
- **Orientado a eventos vs Request/Response (RPC/REST)** = cómo los componentes se _comunican_ a través de los límites que hayas dibujado.

Podés tener un monolito request/response (este proyecto), un monolito orientado a eventos (un solo proceso que se habla a sí mismo vía un bus de eventos en memoria), microservicios request/response (servicios llamando a las APIs REST/gRPC de otros sincrónicamente), o microservicios orientados a eventos (servicios publicando a Kafka/SQS/RabbitMQ, desacoplados). Confundir los dos ejes es el error más común cuando este tema aparece en entrevistas.

<a id="sub-17-1"></a>

### Monolito — cuándo es la decisión correcta

Un monolito es **una unidad desplegable** que contiene toda la lógica de negocio, aunque internamente esté organizada en módulos/dominios (como las carpetas `url/`, `click/`, `counter/` de este proyecto).

**Usá un monolito cuando:**

- El equipo es chico (aproximadamente < 8-10 ingenieros) — el costo principal de los microservicios es el overhead de coordinación, que solo se paga cuando un solo equipo ya no puede ponerse de acuerdo en un solo codebase/cadencia de release.
- Los límites de dominio todavía no están claros o el producto es pre-product-market-fit. Partir servicios por las costuras equivocadas es mucho más caro de deshacer que partir los módulos de un monolito después — los módulos viven en un repo, una transacción, un deploy; deshacer un límite de red significa reescribir contratos, migrar datos y coordinar los calendarios de release de dos equipos.
- Necesitás consistencia transaccional entre entidades (ej. "decrementar inventario Y crear orden" en una transacción ACID). Las transacciones distribuidas entre servicios son difíciles (two-phase commit, sagas) y un monolito obtiene esto gratis vía la transacción de la DB.
- La simplicidad operativa importa más que el escalado independiente — una cosa para desplegar, una cosa para monitorear, un stream de logs, sin service mesh, sin tracing distribuido para debuggear un request.
- Este proyecto es exactamente este caso: un solo proceso Express. `url/`, `click/`, `counter/` son dominios dentro de una unidad de deploy. El `npm run build && node dist/main.js` del README es toda la historia de despliegue. Sin llamadas de red entre servicios, sin la clase de bugs de falla-parcial-entre-servicios.

**Costo de quedarse monolito demasiado tiempo:** los cambios de un solo equipo empiezan a chocar (conflictos de merge, la suite de tests compartida se vuelve lenta), features no relacionadas deben escalar juntas (no podés escalar el path de redirect read-heavy independientemente del path de shorten write-heavy sin escalar el proceso entero), y un bug en un módulo puede crashear el proceso entero para tráfico no relacionado.

<a id="sub-17-2"></a>

### Microservicios — cuándo es la decisión correcta

Los microservicios parten el sistema en servicios **independientemente desplegables**, cada uno dueño de su propio data store, usualmente comunicándose por red (REST/gRPC, sync) o un broker (async).

**Usá microservicios cuando:**

- El escalado independiente es una necesidad real y medida. Ejemplo concreto de la propia forma de este proyecto: el path de redirect (`GET /:shortCode`) recibe ~10x el tráfico del path de shorten (la config de NGINX acá da `/*` 100 req/s vs `/shorten` 10 req/s) — a escala suficientemente grande partirías el "servicio de redirect" del "servicio de shorten" para poder escalar el path de lectura barato y caliente sin pagar capacidad extra en el path de escritura, y viceversa.
- Partes distintas del sistema tienen requerimientos genuinamente distintos de escalado/disponibilidad/tecnología (ej. un servicio write-heavy al que le va bien la consistencia eventual vs. un servicio de facturación que debe ser ACID).
- Múltiples equipos necesitan shippear independientemente sin bloquearse en los trenes de release del otro — el organigrama es el driver real acá (Ley de Conway: la forma del sistema espeja la estructura de comunicación).
- Necesitás aislamiento de fallas: un servicio crasheando no debería tirar funcionalidad no relacionada. En un monolito, una excepción no manejada o una fuga de memoria en el path de analítica puede matar de hambre al path de redirect en el mismo event loop; como servicios separados, que el servicio de analítica se caiga no toca los redirects.
- Necesidades políglotas: ej. un hot path escrito en Go por throughput crudo mientras el resto queda en Node/TS — solo posible a través de un límite de proceso.

**Costo:** las llamadas de red reemplazan las llamadas a función (latencia, falla parcial, retries, idempotencia), cada servicio necesita su propio CI/CD, monitoreo, historia de on-call; debuggear un solo request de usuario ahora significa correlacionar logs/trazas entre servicios (por _esto_ existe el tracing distribuido — sección 12 — es la herramienta que hace debuggeables a los microservicios). La consistencia de datos entre servicios requiere patrones de saga/outbox en vez de una transacción de DB.

<a id="sub-17-3"></a>

### Arquitectura orientada a eventos — cuándo es la decisión correcta

Orientado a eventos = los componentes se comunican **publicando hechos sobre lo que pasó** (eventos) a un broker, y otros componentes **reaccionan** asincrónicamente, en vez de que un componente llame directamente a otro y espere respuesta.

**Usá orientado a eventos cuando:**

- El productor no debería bloquearse en, ni siquiera conocer, a cada consumidor. Ejemplo de este mismo proyecto: el redirect controller hace inserts de clicks fire-and-forget (sección 7) — conceptualmente esto ya es "event-driven-lite": "ocurrió un redirect" es un hecho al que la lógica de click-tracking reacciona sin que la respuesta del redirect lo espere. A mayor escala esta llamada fire-and-forget con `.catch()` se convierte en un evento real (`UrlClicked`) publicado a Kafka/SQS, y podrías agregar consumidores nuevos (detección de fraude, dashboards de analítica en tiempo real, facturación) sin tocar nunca más el redirect controller — esa es la ganancia central: **consumidores nuevos, cero cambios al productor.**
- Necesitás desacoplar servicios con perfiles de uptime/throughput distintos. Una cola/broker absorbe ráfagas — si la DB de analítica está caída o lenta, los eventos se encolan en vez de que el path de redirect falle o se bloquee (el `analyticsClient` de este proyecto con escrituras `w:1` + fire-and-forget es la versión in-process de esta misma idea — sección "Two MongoDB clients" en CLAUDE.md).
- Los workflows naturalmente abarcan múltiples pasos con retries/estado de larga duración (orden creada → pago cobrado → inventario reservado → enviado) — orientado a eventos más una saga/máquina de estados encaja mejor que una cadena larga de llamadas sincrónicas que debe quedarse abierta durante todo eso.
- La auditoría/replay importa: un log de eventos es una historia durable de "todo lo que pasó", que podés reproducir para reconstruir estado, debuggear un incidente, o alimentar un servicio nuevo que ni existía cuando los eventos se produjeron originalmente.

**Costo:** consistencia eventual (el consumidor puede procesar el evento segundos después — bien para "mandar email de bienvenida", mal para "confirmar el pago antes de enviar"), debuggear requiere trazar un evento a través de N consumidores async en vez de leer un call stack lineal, necesitás un broker (Kafka/SQS/RabbitMQ) como infraestructura nueva que correr/monitorear, y la semántica de orden de mensajes/dedup/entrega-at-least-once se vuelven problemas reales que debés diseñar (consumidores idempotentes, claves de dedup).

<a id="sub-17-4"></a>

### Atajo de decisión

| Pregunta                                                                                              | Se inclina hacia                                                                                |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| ¿Equipo chico, límites de dominio poco claros, necesitás transacciones?                               | Monolito                                                                                        |
| ¿Necesitás escalado/deploys independientes por dominio bien entendido, múltiples equipos?             | Microservicios                                                                                  |
| ¿El productor no debería esperar ni conocer a los consumidores; necesitás fan-out, buffering, replay? | Orientado a eventos                                                                             |
| ¿Ninguno de los dolores de arriba existe todavía?                                                     | Monolito (default) — partí cuando aparezca un dolor _específico y medido_, no especulativamente |

El patrón real más fuerte: **empezar monolito (request/response), extraer microservicios solo por las costuras donde realmente sentiste el dolor** (un módulo específico escalando distinto, un equipo específico bloqueado en releases), e **introducir eventos solo donde el desacople específicamente paga** (fan-out a múltiples consumidores futuros desconocidos, absorber picos de carga, necesidades de auditoría/replay) — no como estilo de comunicación default en todos lados.

<a id="sub-17-5"></a>

### Cómo encaja este proyecto

Monolito + mayormente request/response, con un borde async fire-and-forget (los inserts de clicks) que es arquitecturalmente la semilla de un límite orientado a eventos. Si este proyecto necesitara escalar más, las dos particiones más probables, siguiendo la regla de "partir por dolor medido" de arriba, serían:

1. Separar el servicio de redirect del servicio de shorten (perfiles de tráfico distintos, ver los rate limits de NGINX).
2. Convertir el insert de click fire-and-forget en un evento real publicado (`UrlClicked`) para que la analítica de clicks pueda ser consumida por múltiples servicios futuros sin cambios en el path de redirect.

**Referencias:**

- [Martin Fowler: Microservices](https://martinfowler.com/articles/microservices.html)
- [Martin Fowler: MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- [Martin Fowler: What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html)
- [AWS: Monolithic vs Microservices Architecture](https://aws.amazon.com/microservices/)
- [Confluent: Event-Driven Architecture](https://www.confluent.io/learn/event-driven-architecture/)
- [Conway's Law](https://www.melconway.com/Home/Conways_Law.html)

---

<a id="sec-18"></a>

## 18. API Gateway — Qué Es, Patrones, Casos de Uso

<a id="sub-18-1"></a>

### Qué hace realmente un API Gateway

Un **API Gateway** es el punto de entrada único que se ubica entre los clientes y un conjunto de servicios backend, y hace el trabajo transversal **una vez, en el borde**, en vez de que cada servicio lo reimplemente:

```
                          ┌─────────────────────────────┐
cliente ──HTTPS──►        │        API Gateway           │
                          │ ─ Terminación TLS            │
                          │ ─ Ruteo (path → servicio)    │──► servicio A (shorten)
                          │ ─ Auth / API keys / JWT       │──► servicio B (redirect)
                          │ ─ Rate limiting / cuotas      │──► servicio C (analytics)
                          │ ─ Transformación req/res     │──► ...
                          │ ─ Agregación / fan-out        │
                          │ ─ Logging / métricas / trazas │
                          └─────────────────────────────┘
```

La idea central: los clientes ven **una** superficie de API; el gateway esconde cuántos servicios realmente la responden, y dónde viven. Es la misma motivación de "esconder las costuras" detrás de un [facade](https://en.wikipedia.org/wiki/Facade_pattern), aplicada a la capa de red.

**Responsabilidades típicas, de más a menos universales:**

| Responsabilidad        | Qué significa                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------------- |
| Ruteo                  | `/orders/*` → orders-service, `/users/*` → users-service                                    |
| Terminación TLS        | El gateway tiene el certificado; el tráfico interno puede ser HTTP plano en una red privada |
| Rate limiting / cuotas | Límites por cliente o por ruta aplicados antes de que el request llegue al código de la app |
| AuthN/AuthZ            | Validar API keys / JWT / token OAuth una vez, en el borde                                   |
| Transformación req/res | Reescribir headers, remodelar payloads, traducción de protocolo (REST↔gRPC)                 |
| Agregación             | Una llamada del cliente hace fan-out a N llamadas backend, el gateway mergea la respuesta   |
| Observabilidad         | Punto central para emitir access logs / métricas / spans de trazas consistentes             |
| Resiliencia            | Circuit breaking, retries, timeouts aplicados uniformemente en el borde                     |

<a id="sub-18-2"></a>

### API Gateway vs reverse proxy vs load balancer vs service mesh

Estos se confunden constantemente — se solapan pero resuelven problemas de forma distinta:

| Herramienta       | Trabajo primario                                                                                                                                    | Alcance                                     | Ejemplo                              |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------------------ |
| **Reverse proxy** | Reenviar un request a un backend, esconder la topología del backend                                                                                 | Genérico — cualquier protocolo/propósito    | NGINX, HAProxy (crudo)               |
| **Load balancer** | Distribuir requests entre N réplicas de un servicio                                                                                                 | Un servicio, muchas instancias              | NGINX `upstream`, ELB                |
| **API Gateway**   | Punto de entrada único de borde para _muchos servicios distintos_, con preocupaciones específicas de API (auth, cuotas, transformación, agregación) | Toda la superficie de API, muchos servicios | Kong, AWS API Gateway, Envoy Gateway |
| **Service mesh**  | Tráfico este-oeste _entre_ servicios internos (no de cara al cliente)                                                                               | Servicio-a-servicio interno                 | Istio, Linkerd                       |

Un reverse proxy es el _mecanismo_; un API Gateway es un reverse proxy con **políticas con forma de API** encima (auth, cuotas, transformaciones por ruta, agregación). Un load balancer es usualmente un ingrediente _dentro_ de un gateway (rutear a réplicas sanas), no un reemplazo. Un service mesh resuelve la misma clase de preocupaciones transversales (retries, mTLS, observabilidad) pero para el tráfico _entre_ tus propios servicios, no el borde de cara al cliente — los dos se usan frecuentemente juntos: gateway en el borde, mesh internamente.

<a id="sub-18-3"></a>

### Patrones centrales

**Gateway Routing** — rutear por path/host/header al backend correcto, sin agregar nada más:

```
/api/v1/data/shorten  →  shorten-service
/:shortCode            →  redirect-service
```

**Gateway Aggregation** — un request del cliente dispara llamadas a múltiples servicios backend; el gateway (o una capa BFF) mergea los resultados en una respuesta, ahorrándole al cliente N round-trips:

```
GET /dashboard
  el gateway llama: user-service, orders-service, notifications-service (en paralelo)
  el gateway mergea: { user, orders, notifications } → una sola respuesta JSON
```

**Gateway Offloading** — mover las preocupaciones transversales fuera de cada servicio y al gateway una sola vez: terminación TLS, validación de tokens de auth, rate limiting, compresión, headers CORS. Los servicios confían en que el tráfico que les llega ya pasó esos chequeos.

**Backend For Frontend (BFF)** — en vez de un gateway genérico para todos los clientes, correr un gateway _por tipo de cliente_ (web-BFF, mobile-BFF), cada uno moldeando/agregando respuestas de forma distinta para las necesidades de ese cliente (mobile quiere payloads más chicos, web quiere más detalle). Evita que el contrato de un solo gateway se convierta en un compromiso de mínimo común denominador entre clientes muy distintos.

<a id="sub-18-4"></a>

### Cuándo necesitás uno

| Señal                                                                                    | Se inclina hacia                                |
| ---------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Un servicio backend, un tipo de cliente                                                  | Un reverse proxy simple alcanza (este proyecto) |
| Múltiples servicios backend, necesitás una superficie de API pública consistente         | API Gateway                                     |
| Cada servicio reimplementando su propio auth/rate-limit/logging                          | API Gateway (offloadearlo una vez)              |
| El cliente necesita datos ensamblados de varios servicios en un round trip               | API Gateway con agregación, o BFF               |
| Formas de cliente muy distintas (mobile vs web vs API de partners)                       | BFF por tipo de cliente                         |
| Las preocupaciones transversales están entre servicios _internos_, no de cara al cliente | Service mesh, no un gateway                     |

<a id="sub-18-5"></a>

### Implementaciones en producción

| Herramienta                                                                           | Notas                                                                  |
| ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| [Kong](https://konghq.com/products/kong-gateway)                                      | NGINX + ecosistema de plugins Lua; self-hosted o cloud                 |
| [AWS API Gateway](https://aws.amazon.com/api-gateway/)                                | Administrado, se integra con Lambda/ALB; pago por request              |
| [Envoy Gateway](https://gateway.envoyproxy.io/) / [Envoy](https://www.envoyproxy.io/) | Proxy L7 de alto rendimiento; también el data plane dentro de Istio    |
| [Apigee](https://cloud.google.com/apigee)                                             | API management empresarial de Google (analítica, monetización, cuotas) |
| [Traefik](https://traefik.io/traefik/)                                                | Auto-descubre backends vía labels de Docker/Kubernetes                 |
| [Netflix Zuul](https://github.com/Netflix/zuul)                                       | Gateway JVM; mayormente superado por Spring Cloud Gateway              |

<a id="sub-18-6"></a>

### Tradeoffs

| Preocupación            | Sin gateway                                     | Con gateway                                                          |
| ----------------------- | ----------------------------------------------- | -------------------------------------------------------------------- |
| Código transversal      | Duplicado en cada servicio                      | Escrito una vez, centralmente                                        |
| Punto único de falla    | N/A                                             | Gateway caído = todo caído (debe ser HA, replicado)                  |
| Latencia                | Cliente→servicio directo                        | Un salto de red extra                                                |
| Complejidad operativa   | Cada servicio simple                            | La config/deploy del gateway es una cosa más que correr y monitorear |
| Simplicidad del cliente | El cliente debe conocer N ubicaciones/contratos | El cliente ve una superficie de API                                  |

<a id="sub-18-7"></a>

### Cómo encaja este proyecto

NGINX acá (`nginx/nginx.conf`) es un **reverse proxy + offload parcial de gateway**, no un API Gateway completo — hay un solo servicio backend (`app1`), así que el ruteo no tiene nada entre qué rutear. Lo que sí toma, en el borde, coincide con el patrón "Gateway Offloading" de arriba:

- **Rate limiting por ruta** — `limit_req_zone` le da a `/api/v1/data/shorten` (escritura, 10 r/s) un presupuesto distinto que a los redirects `/*` (lectura, 100 r/s) — ver §1 (Rate Limiting L7). Este es exactamente el tipo de política que un gateway aplica una vez en el borde en vez de que cada handler ruede su propio limiter.
- **Modelado uniforme de errores** — `error_page 429 /429.json` devuelve un body de error JSON consistente sin importar qué location upstream matcheó.
- **Inyección de headers** — `X-Real-IP` / `X-Forwarded-For` seteados una vez, así `app1` no necesita su propia lógica de proxy-trust por ruta.

Lo que deliberadamente **no** hace (porque todavía no lo necesita): validación de auth/API-keys, agregación de requests, traducción de protocolos, o modelado de respuestas por cliente — esos solo ganan su complejidad cuando hay más de un servicio backend o tipo de cliente que reconciliar (la regla de §17 de "partir por dolor medido" aplica acá también). Si este proyecto creciera hacia la partición en microservicios descrita en §17 (shorten-service / redirect-service separados), el rol de NGINX crecería naturalmente de "reverse proxy con rate limiting offloadeado" a un API Gateway real: ruteando por path a dos upstreams distintos, unificando auth entre ambos, y potencialmente agregando un endpoint `/stats/:shortCode` desde ambos servicios en una respuesta.

**Referencias:**

- [Microsoft: API Gateway pattern](https://learn.microsoft.com/en-us/azure/architecture/microservices/design/gateway)
- [Microsoft: Backends for Frontends pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/backends-for-frontends)
- [NGINX: API Gateway](https://www.nginx.com/learn/api-gateway/)
- [Kong: What is an API Gateway](https://konghq.com/learning-center/api-gateway/what-is-an-api-gateway)
- [AWS: What is an API Gateway](https://aws.amazon.com/api-gateway/)
- [Envoy Gateway docs](https://gateway.envoyproxy.io/)
- [Sam Newman: Backends For Frontends](https://samnewman.io/patterns/architectural/bff/) (citado ampliamente por el sitio de Fowler)

---

<a id="sec-19"></a>

## 19. SQL vs NoSQL — Qué Es Cada Uno y Cuándo Usarlos

### Qué es cada uno

**SQL (relacional):** datos modelados como **tablas con esquema fijo** (columnas tipadas), relacionadas entre sí por claves foráneas. Se consulta con SQL, que permite **JOINs** arbitrarios entre tablas. Ejemplos: PostgreSQL, MySQL, SQL Server, Oracle.

**NoSQL:** paraguas para todo lo que no es relacional. No es una sola cosa — son cuatro familias con tradeoffs muy distintos:

| Familia            | Modelo                                   | Ejemplos                     | Caso típico                                 |
| ------------------ | ---------------------------------------- | ---------------------------- | ------------------------------------------- |
| **Documento**      | JSON/BSON anidado, esquema flexible      | MongoDB, CouchDB             | Entidades autocontenidas (perfil, catálogo) |
| **Clave/valor**    | `clave → blob opaco`                     | Redis, DynamoDB (en su base) | Cache, sesiones, lookups por ID             |
| **Columnar ancho** | Filas con millones de columnas dispersas | Cassandra, HBase             | Series temporales, escrituras masivas       |
| **Grafo**          | Nodos + aristas                          | Neo4j, Neptune               | Relaciones profundas (red social, fraude)   |

### La diferencia estructural real

- **SQL:** el esquema y las relaciones viven en la base. La base garantiza integridad (constraints, FKs) y te da **transacciones ACID multi-fila/multi-tabla** gratis. Escalás principalmente **vertical** (server más grande) o con read-replicas; el sharding es posible pero doloroso.
- **NoSQL:** el "esquema" vive en el código de aplicación. Se renuncia a JOINs y a transacciones globales a cambio de **particionado horizontal nativo**: los datos se distribuyen por clave entre muchos nodos, y escalar = agregar nodos. La consistencia suele ser configurable (fuerte ↔ eventual — el write concern de Mongo en §4 es exactamente esa perilla).

**ACID vs BASE:** ACID = Atomicity, Consistency, Isolation, Durability (el default relacional). BASE = Basically Available, Soft state, Eventually consistent (el default NoSQL distribuido). No es que NoSQL "no tenga transacciones" (Mongo tiene transacciones multi-documento desde 4.0) — es que su modelo de costos las desalienta: si necesitás transacciones entre entidades todo el tiempo, elegiste mal la herramienta.

### Sobre el apunte "SQL => write heavy, MongoDB => read heavy"

**Cuidado: así como está, es incorrecto como regla general** — y en una entrevista te lo van a repreguntar. La elección no la define read-vs-write-heavy sino **el modelo de datos y el patrón de acceso**:

- Un SQL bien indexado maneja cargas read-heavy enormes (agregale read-replicas y un cache delante).
- Cassandra/DynamoDB son famosas justamente por cargas **write-heavy** masivas — o sea, NoSQL puede ser la opción write-heavy.
- MongoDB rinde muy bien en lecturas cuando el documento coincide con lo que la app necesita leer (todo en un solo fetch, sin JOINs) — de ahí sale la intuición "Mongo = read heavy", pero la causa real es **localidad del dato**, no "NoSQL lee más rápido".

La pregunta correcta en una entrevista: _"¿Cómo se accede al dato?"_

- ¿Consultas ad-hoc, reportes, relaciones entre muchas entidades, integridad fuerte? → **SQL**.
- ¿Lookups por clave conocida, a escala, con esquema que evoluciona rápido y datos autocontenidos? → **documento/clave-valor**.
- ¿Ingesta masiva append-only con lecturas por rango de tiempo? → **columnar**.

### Cómo encaja este proyecto

Este proyecto usa MongoDB con un patrón de acceso 100% clave→valor (`shortUrl → longUrl`, `longUrlHash → doc`): nunca hay JOINs, el documento es autocontenido, y el path caliente es un lookup por índice único. Es el caso de libro para documento/clave-valor. Un Postgres con una tabla de dos columnas también funcionaría perfectamente a esta escala — la elección se vuelve estructural recién cuando el sharding horizontal es necesidad real.

**Referencias:**

- [Martin Kleppmann: Designing Data-Intensive Applications — cap. 2 (modelos de datos)](https://dataintensive.net/)
- [MongoDB: SQL vs NoSQL](https://www.mongodb.com/nosql-explained/nosql-vs-sql)
- [AWS: ¿Qué es NoSQL?](https://aws.amazon.com/nosql/)
- [Martin Fowler: NosqlDistilled](https://martinfowler.com/books/nosql.html)

---

<a id="sec-20"></a>

## 20. DynamoDB

**DynamoDB** es la base clave/valor y de documentos **totalmente administrada** de AWS. Vos no ves servers, réplicas ni shards: definís tablas, y AWS particiona y replica por debajo (multi-AZ) con latencia de un dígito de milisegundos a prácticamente cualquier escala.

### El modelo de datos (lo que lo define)

- **Partition key (PK):** obligatoria. DynamoDB hashea la PK para decidir en qué partición física vive el ítem. Todo acceso eficiente empieza por conocer la PK.
- **Sort key (SK):** opcional. Dentro de una partición, los ítems se ordenan por SK — habilita queries por rango (`begins_with`, `between`) dentro de esa partición.
- **Sin JOINs, sin queries ad-hoc:** solo podés consultar eficientemente por PK (+ condiciones de SK). Un `Scan` de tabla completa existe pero es carísimo y lento — si lo necesitás seguido, elegiste mal la herramienta o el diseño.
- **Índices secundarios:** GSI (Global Secondary Index) te da una PK/SK alternativa (réplica interna con consistencia eventual); LSI comparte la PK con otra SK.

**Consecuencia de diseño:** en DynamoDB **primero se listan los patrones de acceso, después se diseña la tabla** (a menudo "single-table design": varias entidades en una tabla, con PKs/SKs compuestas tipo `USER#123` / `ORDER#2024-06-30`). Lo inverso al modelado relacional, donde normalizás primero y consultás después.

### Capacidad y costos

- **On-demand:** pagás por request (RCU/WCU consumidas). Cero planificación, ideal para tráfico impredecible.
- **Provisioned:** reservás capacidad de lectura/escritura por segundo (más barato si el tráfico es estable; con auto-scaling).
- **Hot partition:** si una sola PK concentra el tráfico (ej. `PK = "config"` global), esa partición se satura aunque la tabla tenga capacidad de sobra — el equivalente Dynamo de la bomba de cardinalidad: la clave debe distribuir bien.

### Features que aparecen en entrevistas

| Feature              | Qué es                                                                                                                                   |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| **DynamoDB Streams** | Log de cambios (CDC) de la tabla — dispara Lambdas por cada insert/update/delete; es el hook natural hacia arquitectura de eventos (§28) |
| **TTL**              | Expiración automática de ítems por atributo timestamp — gratis, para sesiones/tokens                                                     |
| **Transacciones**    | `TransactWriteItems` — ACID hasta 100 ítems, con costo doble                                                                             |
| **DAX**              | Cache in-memory administrado delante de Dynamo (microsegundos) — el "Redis administrado" de Dynamo                                       |
| **Consistencia**     | Lecturas eventually consistent por defecto; strongly consistent bajo demanda (2× costo, solo sobre la tabla base, no GSIs)               |

### Cuándo usarlo / cuándo no

- **Sí:** patrones de acceso conocidos y estables por clave, escala grande o spiky, cero ganas de operar infraestructura, integración serverless (Lambda).
- **No:** queries ad-hoc/analíticas (eso es para un warehouse u OpenSearch), relaciones complejas, o si el equipo va a querer cambiar los patrones de acceso todo el tiempo.

**Para este proyecto:** `shortUrl → longUrl` es literalmente el caso de uso ideal de DynamoDB (lookup por clave exacta, read-heavy, escala). La versión AWS-native de este proyecto sería API Gateway + Lambda + DynamoDB (con DAX o sin cache siquiera) — sin Mongo, sin Redis, sin servers.

**Referencias:**

- [AWS: DynamoDB — cómo funciona](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.html)
- [The DynamoDB Book — Alex DeBrie (single-table design)](https://www.dynamodbbook.com/)
- [AWS: paper original de Dynamo (2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Alex DeBrie: single-table design](https://www.alexdebrie.com/posts/dynamodb-single-table/)

---

<a id="sec-21"></a>

## 21. OpenSearch

**OpenSearch** es un motor de **búsqueda y analítica** distribuido, fork open-source de Elasticsearch (AWS lo forkeó en 2021 cuando Elastic cambió de licencia). Mismo modelo mental que Elasticsearch: documentos JSON indexados en un **índice invertido**.

### El índice invertido (la idea central)

Una base normal indexa `id → documento`. Un índice invertido indexa `término → lista de documentos que lo contienen`:

```
doc1: "el gato negro"        índice invertido:
doc2: "el perro negro"         "gato"  → [doc1]
                               "perro" → [doc2]
                               "negro" → [doc1, doc2]
```

Por eso buscar "negro" en 100 millones de documentos es O(1) al término + merge de listas, y no un scan. Sumale **análisis de texto** (tokenización, minúsculas, stemming: "corriendo" → "corr", sinónimos, tolerancia a typos con fuzziness) y **scoring de relevancia** (BM25) — cosas que un `LIKE '%negro%'` de SQL o un `$regex` de Mongo no pueden hacer eficientemente ni con calidad de ranking.

### Para qué se usa

| Caso                    | Por qué OpenSearch                                                                                        |
| ----------------------- | --------------------------------------------------------------------------------------------------------- |
| **Full-text search**    | Buscador de productos/documentos con relevancia, typos, facetas                                           |
| **Log analytics**       | El stack ELK/OpenSearch: Logstash/Fluentd → OpenSearch → Dashboards (la alternativa pesada a Loki de §12) |
| **Agregaciones**        | Facetas y métricas sobre millones de docs en tiempo casi real                                             |
| **Observabilidad/SIEM** | Trazas, eventos de seguridad, detección de anomalías                                                      |

### La regla de oro: no es fuente de verdad

OpenSearch es casi siempre un **índice derivado**, igual que Redis es un cache derivado en este proyecto: la fuente de verdad vive en SQL/Mongo/Dynamo, y un pipeline (CDC, DynamoDB Streams, cola de eventos) replica los cambios hacia OpenSearch. Es **near-real-time** (los documentos son buscables ~1s después de indexarse, no inmediatamente) y su durabilidad/consistencia no es la de una base transaccional.

**Patrón típico completo (y respuesta de entrevista):**

```
escritura → base transaccional (fuente de verdad)
              └─► evento/CDC ─► OpenSearch (búsqueda)  ─► queries de texto/facetas
                             └─► Redis (cache)          ─► lookups calientes por clave
```

**Para este proyecto:** si hubiera que agregar "buscá entre tus URLs por palabras del título de la página destino", eso es OpenSearch — Mongo con `$regex` no escala para eso ni rankea relevancia.

**Referencias:**

- [OpenSearch docs](https://opensearch.org/docs/latest/)
- [Elastic: inverted index](https://www.elastic.co/guide/en/elasticsearch/guide/current/inverted-index.html)
- [AWS: OpenSearch Service](https://aws.amazon.com/opensearch-service/)
- [BM25 — el algoritmo de relevancia](https://en.wikipedia.org/wiki/Okapi_BM25)

---

<a id="sec-22"></a>

## 22. Redis a Fondo — Clave/Valor, Cache, Idempotencia y Atomicidad

### Qué es

Redis es un **almacén clave/valor in-memory**: toda la data vive en RAM (de ahí la latencia sub-ms), con persistencia opcional a disco (AOF/RDB, ver §7). El valor no es solo un string — tiene estructuras de datos nativas: strings, hashes, lists, sets, sorted sets, streams (§7), HyperLogLog, bitmaps.

### Qué es un cache (la definición de entrevista)

> Un cache almacena un dato de forma **temporal**, en un medio más rápido que la fuente, para evitar recomputarlo o volver a buscarlo. Es **derivado, no fuente de verdad**: perderlo cuesta latencia, nunca correctitud.

Las tres decisiones que definen un cache:

1. **Estrategia de llenado:** cache-aside (la app busca en cache, si miss va a la DB y puebla — lo más común), write-through (se escribe en cache y DB juntas — lo que hace este proyecto al acortar), write-behind (se escribe al cache y la DB se actualiza async — rápido pero riesgoso).
2. **Estrategia de expiración:** TTL (tiempo fijo) y/o evicción por presión de memoria (LFU/LRU — §5).
3. **Invalidación:** el problema famoso. Si el dato de origen cambia, ¿quién actualiza/borra la clave? (En este proyecto no existe el problema: las URLs son inmutables — por eso el diseño es tan simple.)

### Redis como control de idempotencia

**Idempotencia** = ejecutar la misma operación N veces produce el mismo resultado que ejecutarla una vez. Crítico en §17/§28: con colas at-least-once y retries, **todo consumidor/endpoint de escritura va a recibir duplicados** — la pregunta no es si, sino cuándo.

El patrón con Redis: el cliente manda una clave de idempotencia (ej. header `Idempotency-Key: uuid`), y el server intenta registrarla **atómicamente** antes de procesar:

```
SET idem:{uuid} "processing" NX EX 86400
```

- `NX` = solo setear si **no existe** (not exists). Si ya existía, `SET` devuelve nil → es un duplicado → devolver la respuesta guardada del primer intento (o 409), sin re-procesar.
- `EX 86400` = TTL de 24h — la ventana de dedup. Sin TTL, las claves se acumulan para siempre.
- La operación completa es **un solo comando atómico**: no hay carrera entre "chequear si existe" y "marcarla" — dos requests concurrentes con el mismo uuid, uno gana, el otro ve el duplicado. Un `GET` seguido de `SET` en dos pasos tendría exactamente esa carrera (ver siguiente subsección).

Esto es lo que resuelve el clásico "el usuario tocó dos veces el botón de pagar" y el "SQS me entregó el mensaje dos veces".

### Atomicidad: por qué "un script" y no GET + SET

Redis ejecuta comandos en **un solo thread**: cada comando individual es atómico por construcción. Pero **una secuencia de comandos desde el cliente no lo es** — entre tu `GET` y tu `SET` puede colarse otro cliente:

```
❌ Carrera clásica (check-then-act en dos pasos):
cliente A: GET counter → 5
cliente B: GET counter → 5
cliente A: SET counter 6
cliente B: SET counter 6      ← se perdió un incremento
```

Tres formas de recuperar la atomicidad (esto es lo que estaba detrás del apunte "Redis se invoca por un script, operación atómica"):

1. **Comandos atómicos de un paso** — si existe uno que hace lo tuyo, usalo: `INCR` (contador atómico), `SET ... NX` (lock/idempotencia), `GETDEL`, `SETNX`. El caso de arriba se arregla con `INCR counter`.
2. **Scripts Lua (`EVAL`)** — para lógica compuesta. Redis ejecuta el script **entero sin intercalar ningún otro comando**: leer-decidir-escribir se vuelve una unidad atómica. Ejemplo real, rate limiter de ventana fija:

   ```lua
   -- KEYS[1] = clave del contador, ARGV[1] = límite, ARGV[2] = ttl
   local current = redis.call('INCR', KEYS[1])
   if current == 1 then
     redis.call('EXPIRE', KEYS[1], ARGV[2])
   end
   if current > tonumber(ARGV[1]) then
     return 0  -- rechazado
   end
   return 1    -- permitido
   ```

   `INCR` + `EXPIRE` + comparación, sin carreras posibles. Así implementan sus operaciones atómicas la mayoría de las librerías de rate limiting y locking sobre Redis (Redlock incluido).

3. **MULTI/EXEC (transacciones)** — encola comandos y los ejecuta en bloque; con `WATCH` da optimistic locking. Menos flexible que Lua (no podés decidir con el valor leído a mitad de camino sin `WATCH`+retry).

**Nota este-proyecto:** el contador de IDs acá usa el `$inc` atómico de **MongoDB** (mismo concepto: la atomicidad la da el datastore, no el cliente). La versión Redis sería `INCR`.

### Escalado horizontal de Redis

"Escalado horizontal = más servidores" (ver §27). Para Redis concretamente:

- **Replicación (primario + réplicas):** escala **lecturas** y da failover (con Sentinel), pero todas las escrituras siguen yendo al primario.
- **Redis Cluster:** escala **escrituras y memoria** particionando el keyspace en **16384 hash slots** repartidos entre nodos: `slot = CRC16(key) mod 16384`. Cada nodo es dueño de un rango de slots; el cliente (cluster-aware) rutea cada clave a su nodo. Consecuencia importante: operaciones multi-clave (y scripts Lua) solo funcionan si todas las claves caen en el mismo slot — se fuerza con **hash tags**: `user:{123}:profile` y `user:{123}:cart` (se hashea solo lo que está entre `{}`).
- Este proyecto usa una sola instancia: correcto para el alcance, y el circuit breaker (§3) cubre su indisponibilidad.

**Referencias:**

- [Redis: SET (opciones NX/EX)](https://redis.io/commands/set/)
- [Redis: programación con Lua](https://redis.io/docs/interact/programmability/eval-intro/)
- [Redis: transacciones MULTI/EXEC](https://redis.io/docs/interact/transactions/)
- [Redis Cluster specification](https://redis.io/docs/reference/cluster-spec/)
- [Stripe: claves de idempotencia](https://stripe.com/docs/api/idempotent_requests)

---

<a id="sec-23"></a>

## 23. Pool de Conexiones

### Qué es

Un **pool de conexiones** es un conjunto de conexiones **abiertas y reutilizables** a un recurso (base de datos, Redis, HTTP keep-alive). Abrir una conexión es caro — handshake TCP + TLS + autenticación + asignación de memoria en el server (~decenas de ms y varios MB del lado de la DB) — así que en vez de abrir/cerrar por request, la app pide una conexión prestada al pool, la usa, y la devuelve.

```
sin pool:  request → [abrir conexión 20ms] → query 2ms → [cerrar] → respuesta   (dominado por overhead)
con pool:  request → [pedir al pool ~0ms] → query 2ms → [devolver] → respuesta
```

### La cuenta que importa con escalado horizontal

Esto es lo que estaba detrás del apunte: el pool se configura **por instancia**, pero el recurso lo ve **multiplicado**:

```
pool_size = 30 por instancia
10 instancias de la app (auto-scaling)  →  300 conexiones contra la DB
la DB acepta max_connections = 200      →  💥 "too many connections" justo en el pico de tráfico
```

O sea: **escalar horizontalmente la app multiplica las conexiones contra los recursos compartidos.** El límite de la base es global; tu pool es local. Con serverless (una Lambda por request) el problema explota — por eso existen los **proxies de conexión**: PgBouncer (Postgres), RDS Proxy (AWS), que mantienen un pool chico real contra la DB y multiplexan miles de clientes sobre él.

Regla práctica: `pool_por_instancia × instancias_máximas < max_connections_de_la_DB`, con margen para réplicas, migraciones y humanos con un cliente SQL abierto. Y contraintuitivo pero cierto: **pools más chicos suelen rendir más** — una DB con 8 cores no ejecuta 300 queries a la vez; cientos de conexiones solo agregan context-switching (la fórmula de partida de HikariCP: `conexiones ≈ cores × 2 + discos`).

### Señales operativas (USE, §9)

- **Utilización:** conexiones en uso / tamaño del pool.
- **Saturación:** requests **esperando** una conexión libre (wait queue del pool) — la señal temprana de que el pool quedó chico o de que hay queries lentas reteniendo conexiones.
- **Errores:** timeouts al adquirir conexión.

El escenario de cascada de §3 empieza exactamente acá: dependencia lenta → conexiones retenidas más tiempo → pool agotado → requests nuevos fallan al instante.

**En este proyecto:** el driver de MongoDB para Node mantiene un pool interno por `MongoClient` (default `maxPoolSize: 100`); hay **dos** clientes (main + analytics), o sea dos pools separados — una falla/saturación del pool de analítica no roba conexiones al path principal. ioredis usa **una sola conexión** multiplexada (Redis es single-threaded; el pipelining sobre una conexión alcanza casi siempre).

**Referencias:**

- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [MongoDB Node driver: connection pool](https://www.mongodb.com/docs/drivers/node/current/fundamentals/connection/connection-options/)
- [PgBouncer](https://www.pgbouncer.org/)
- [AWS RDS Proxy](https://aws.amazon.com/rds/proxy/)

---

<a id="sec-24"></a>

## 24. Hash vs Cifrado — Simétrico y Asimétrico

### Hash ≠ cifrado (la distinción que define la entrevista)

| Propiedad             | **Hash**                                               | **Cifrado**                                    |
| --------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| Dirección             | **Una vía** — no se puede revertir                     | **Dos vías** — se descifra con la clave        |
| Salida                | Tamaño fijo (SHA-256 → 32 bytes) sin importar el input | Tamaño proporcional al input                   |
| Necesita clave        | No (el hash con clave es HMAC)                         | Sí — la seguridad vive en la clave             |
| Pregunta que responde | "¿Es este dato **el mismo** que aquel?"                | "¿Cómo **oculto** esto y lo recupero después?" |

- **Hash = huella digital (fingerprint):** identifica de forma única un contenido sin revelar el contenido. Como la huella en el legajo: te identifica, pero de la huella no se reconstruye la persona. Ejemplos: SHA-256, SHA-3. Usos: integridad de archivos (checksums), dedup (este proyecto hashea `longUrl` con SHA-256 justamente para eso — §15), firma de contenido, claves de cache.
- **Cifrado = ocultar información para recuperarla:** el dato original vuelve a existir aplicando la clave. Usos: datos en tránsito (TLS), datos en reposo (discos, campos sensibles en DB), mensajes.

### ⚠️ Corrección del apunte: las contraseñas se HASHEAN, no se cifran

El apunte decía "encrypt = ocultar info para recuperarla (contraseña)" — **al revés: la contraseña es el ejemplo canónico de hash, no de cifrado.** El servidor **nunca debe poder recuperar** tu contraseña: si la cifrara, la clave de descifrado estaría en algún lado, y quien robe la base + la clave obtiene todas las contraseñas en claro. En cambio se guarda un hash, y en el login se hashea lo que el usuario tipeó y se comparan los hashes.

Y no un hash cualquiera: SHA-256 es **demasiado rápido** para contraseñas (una GPU prueba miles de millones por segundo). Se usan hashes **deliberadamente lentos y con salt**: **bcrypt, scrypt, Argon2** (el salt aleatorio por usuario impide tablas precalculadas/rainbow tables y hace que dos usuarios con la misma contraseña tengan hashes distintos).

```
registro:  "hunter2" + salt → bcrypt (cost 12, ~250ms) → "$2b$12$N9qo8u..." → guardado
login:     bcrypt("hunter2", salt del registro) == hash guardado → ✓
robo de DB: el atacante tiene hashes lentos + salteados → cada contraseña cuesta fuerza bruta individual
```

(Este es también el caso del thread pool de libuv de §7: `bcrypt.hash()` es CPU-bound y corre en los 4 workers.)

### Cifrado simétrico

**Una misma clave** cifra y descifra. Ejemplo: **AES-256-GCM** (el estándar actual; GCM además autentica — detecta manipulación).

- ✅ Rapidísimo (aceleración por hardware AES-NI), sirve para gigabytes.
- ❌ El **problema de distribución de claves**: ambas partes necesitan la misma clave — ¿cómo se la pasás por un canal inseguro?

### Cifrado asimétrico

**Par de claves matemáticamente ligadas**: lo que cifra la **pública** solo lo descifra la **privada** (y viceversa, para firmas). Ejemplos: **RSA**, curvas elípticas (**ECDSA/Ed25519** para firma, **ECDH** para intercambio de claves).

- ✅ Resuelve la distribución: publicás la clave pública; solo vos tenés la privada. Habilita además **firma digital** (cifrar un hash con tu privada → cualquiera verifica con tu pública que fuiste vos y que no se modificó).
- ❌ Lento (~100-1000× vs simétrico) y con límites de tamaño de mensaje.

### Cómo se combinan en la práctica (TLS — la respuesta completa de entrevista)

Ningún sistema real elige uno u otro: usan **cifrado híbrido**.

```
handshake TLS:
  1. asimétrico — el server prueba identidad con su certificado (firma) y las partes
     acuerdan un secreto compartido (ECDH)                         ← caro, una sola vez
  2. simétrico  — todo el tráfico de la sesión se cifra con AES usando ese secreto  ← barato, todo el resto
```

Mismo patrón en SSH, Signal/WhatsApp, PGP. Y JWT usa exactamente la misma dicotomía para sus firmas: HMAC = simétrico, RSA/ECDSA = asimétrico (→ §25).

**Referencias:**

- [OWASP: Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [OWASP: Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [Cloudflare: What is TLS?](https://www.cloudflare.com/learning/ssl/transport-layer-security-tls/)
- [Argon2 — ganador de la Password Hashing Competition](https://github.com/P-H-C/phc-winner-argon2)

---

<a id="sec-25"></a>

## 25. JWT — Autenticación, Autorización y el Chequeo `sub` == `_id`

### Qué es

Un **JWT (JSON Web Token)** es un token **autocontenido y firmado**: lleva sus claims adentro, y la firma permite verificarlas sin consultar una base. Tres partes en base64url separadas por puntos:

```
header.payload.signature

header:    { "alg": "HS256", "typ": "JWT" }
payload:   { "sub": "665f1c...", "role": "user", "iat": 1719840000, "exp": 1719843600 }
signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret)
```

**Claims estándar:** `sub` (subject — el ID del usuario), `exp` (expiración), `iat` (emitido en), `iss` (emisor), `aud` (audiencia).

**Punto crítico:** el payload está **codificado, no cifrado** — cualquiera que tenga el token puede leerlo (base64 decode). La firma garantiza **integridad** (nadie lo modificó) y **autenticidad** (lo emitió quien tiene la clave), no confidencialidad. Nunca pongas secretos en un JWT.

### Firma: simétrica vs asimétrica (conecta con §24)

| Algoritmo         | Tipo       | Quién puede verificar                               | Cuándo                                                                                                                             |
| ----------------- | ---------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **HS256** (HMAC)  | Simétrico  | Solo quien tiene el **mismo secreto** que firmó     | Monolito o servicios que comparten el secreto (riesgo)                                                                             |
| **RS256 / ES256** | Asimétrico | **Cualquiera** con la clave pública (JWKS endpoint) | Microservicios: el auth-service firma con la privada; cada servicio (o el gateway) verifica con la pública sin poder emitir tokens |

En microservicios, RS256 es el default sensato: comprometer un servicio lector no permite falsificar tokens.

### El chequeo "el JWT coincide con el `_id`" (autorización a nivel recurso)

Esto es lo que estaba detrás del apunte. Verificar la firma solo responde **autenticación** ("este token es válido y es de este usuario"). Falta la **autorización sobre el recurso**: ¿este usuario puede tocar **este** documento?

```typescript
// GET /users/:id/orders
const payload = verifyJwt(req.headers.authorization); // autenticación: firma + exp válidas

if (payload.sub !== req.params.id) {
  // autorización: ¿es SU recurso?
  throw new ForbiddenError(); // 403 — token válido, recurso ajeno
}
```

Saltearse ese `if` es la vulnerabilidad **IDOR / Broken Object Level Authorization** — la **#1 del OWASP API Security Top 10**: un token válido de un usuario cualquiera + iterar IDs en la URL = leer los datos de todos. El token válido no implica permiso sobre el recurso; la comparación `payload.sub === recurso.ownerId` (o un chequeo de rol/ownership en la query: `find({ _id, ownerId: sub })`) es lo que cierra el agujero.

### JWT y el API Gateway (conecta con §18)

Del apunte "toda la auth se puede centralizar en el gateway / prevalidar el usuario": el patrón estándar es que el **gateway valide el JWT una vez en el borde** (firma, `exp`, `iss`, `aud`) y rechace ahí lo inválido — los servicios internos reciben solo tráfico pre-autenticado (a menudo con los claims re-inyectados como headers). **Pero** la autorización a nivel recurso (`sub` == `_id`) **no puede** vivir en el gateway — el gateway no sabe de quién es cada documento; eso es lógica de negocio de cada servicio.

### Tradeoffs de JWT vs sesiones en server

- ✅ **Stateless**: cualquier réplica/servicio verifica sin ir a una DB de sesiones — clave para escalado horizontal (§27) y microservicios.
- ❌ **No se puede revocar** antes de `exp` (está firmado, no registrado). Mitigaciones: `exp` corto (5–15 min) + **refresh token** revocable en DB, o una denylist en Redis (`SET revoked:{jti} 1 EX ttl` — idempotencia/§22 otra vez) — lo cual reintroduce estado, que era lo que JWT evitaba. Ese círculo es una pregunta de entrevista clásica.

**Referencias:**

- [RFC 7519 — JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519)
- [jwt.io — debugger de tokens](https://jwt.io/)
- [OWASP API Security Top 10: Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [Auth0: refresh tokens](https://auth0.com/docs/secure/tokens/refresh-tokens)

---

<a id="sec-26"></a>

## 26. Service Mesh y el Patrón Mediator

### El patrón Mediator (GoF, nivel código)

Problema: N objetos que se hablan directamente entre sí = N² referencias cruzadas y acoplamiento total. El **Mediator** introduce un objeto central a través del cual todos se comunican: cada componente conoce **solo al mediador**.

```
sin mediator:  A ↔ B, A ↔ C, B ↔ C, B ↔ D...   (malla, N²)
con mediator:  A ↔ M ↔ B                         (estrella, N)
               C ↔ M ↔ D
```

Ejemplos: una torre de control (los aviones no se coordinan entre sí), un chat room (los usuarios publican al room, no par a par), y en backend librerías como **MediatR** (.NET), donde los controllers despachan comandos/queries a handlers sin conocerlos.

- ✅ Desacopla pares; la lógica de coordinación vive en un solo lugar.
- ❌ El mediador puede volverse un "god object" que concentra toda la lógica.

**La misma idea a distintas escalas** (por esto los apuntes tenían mediator, gateway y mesh juntos): un **event broker** (§17/§28) es un mediator entre servicios — publican al broker, no entre sí. Un **API Gateway** (§18) es un mediator entre clientes y servicios. Un **service mesh** media la comunicación interna. Es el mismo movimiento estructural: reemplazar malla N² por un intermediario.

### Service Mesh (nivel infraestructura)

Del apunte: "políticas entre servers que se consultan" — exacto. Un **service mesh** es una capa de infraestructura que maneja la comunicación **servicio-a-servicio** (este-oeste) en un sistema de microservicios, sin que los servicios lleven esa lógica en su código.

**Cómo funciona — el sidecar proxy:** junto a cada instancia de servicio se despliega un proxy (Envoy, típicamente). Todo el tráfico entrante y saliente del servicio pasa por su sidecar:

```
servicio A ──► [sidecar A] ═══ red (mTLS) ═══ [sidecar B] ──► servicio B
                    ▲                              ▲
                    └────── control plane (Istio) ─┘   ← distribuye políticas/config
```

- **Data plane** = los sidecars (mueven los bytes y aplican las políticas).
- **Control plane** (Istio, Linkerd) = donde declarás las políticas; las distribuye a todos los sidecars.

**Qué políticas aplica** (todo lo que cada servicio tendría que reimplementar, hecho una vez en infraestructura):

| Categoría          | Ejemplos                                                                                                                                         |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Seguridad**      | **mTLS** automático entre todos los servicios (cifrado + identidad mutua); políticas de autorización ("payments solo acepta llamadas de orders") |
| **Resiliencia**    | Retries con backoff, timeouts, **circuit breakers** (§3) — por config, no por código                                                             |
| **Tráfico**        | Canary/traffic splitting ("5% de las llamadas a la v2"), mirroring, load balancing                                                               |
| **Observabilidad** | Métricas RED (§9), trazas distribuidas (§12) y access logs uniformes para todo el tráfico interno, gratis                                        |

**Gateway vs mesh (la distinción de entrevista, ya tabulada en §18):** el gateway gobierna el tráfico **norte-sur** (clientes externos → sistema); el mesh gobierna el **este-oeste** (servicios entre sí). Se complementan: gateway en el borde, mesh adentro. De hecho el mismo proxy (Envoy) suele ser el data plane de ambos.

**Cuándo lo justificás:** con muchos microservicios (decenas+), donde reimplementar mTLS/retries/observabilidad en cada servicio (y cada lenguaje) no escala. Para pocos servicios, el mesh es overhead operativo enorme — este proyecto (un servicio) está a dos órdenes de magnitud de necesitarlo.

**Referencias:**

- [Patrón Mediator — Refactoring Guru](https://refactoring.guru/design-patterns/mediator)
- [Istio: What is a service mesh?](https://istio.io/latest/about/service-mesh/)
- [Linkerd: service mesh 101](https://linkerd.io/what-is-a-service-mesh/)
- [Envoy Proxy](https://www.envoyproxy.io/)
- [NGINX: What Is a Service Mesh?](https://www.nginx.com/blog/what-is-a-service-mesh/)

---

<a id="sec-27"></a>

## 27. Escalado Horizontal vs Vertical

### Las dos direcciones

| Dirección                  | Qué hacés                                             | Límite                                                                                                         |
| -------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **Vertical** (scale up)    | Server más grande: más CPU/RAM/disco                  | Techo físico de hardware, precio exponencial, sigue siendo **un** punto de falla, escalar = downtime/migración |
| **Horizontal** (scale out) | **Más servidores** iguales detrás de un load balancer | Coordinación: el sistema debe estar diseñado para ello                                                         |

(El apunte decía "más cables" — más **servidores**. Aunque cables también vas a necesitar.)

Vertical es más simple y suele ser el primer paso correcto (nada que rediseñar). Horizontal es lo que da **elasticidad** (agregar/sacar instancias según carga) y **disponibilidad** (una instancia muere → el LB la saca del pool y el resto absorbe, ver §10 readiness).

### El requisito: instancias stateless

Escalar horizontalmente funciona solo si **cualquier instancia puede atender cualquier request**. Todo estado debe salir del proceso y mudarse a almacenes compartidos:

| Estado en el proceso            | A dónde va                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------- |
| Sesiones en memoria             | Redis, o tokens autocontenidos (JWT — §25)                                                   |
| Archivos subidos al disco local | Object storage (S3)                                                                          |
| Cache in-process                | Cache compartido (Redis — §22) — si no, cada instancia tiene su propia versión inconsistente |
| Contadores/locks en memoria     | Operaciones atómicas en el datastore (`$inc`, `INCR` — §22)                                  |
| Cron jobs dentro de la app      | Un scheduler externo o elección de líder — o corren N veces                                  |

Las _sticky sessions_ (el LB fija cada usuario a una instancia) son el parche cuando hay estado local — frágil: se pierde al escalar hacia abajo o al morir la instancia.

### Efectos derrame (todo conecta)

- **Pool de conexiones (§23):** N instancias × pool = presión multiplicada sobre la DB — el límite se corre, no desaparece.
- **Thundering herd (§6):** N instancias reconectando a la vez al mismo recurso.
- **Rate limiting (§1):** un límite en memoria por instancia deja de ser global — o lo aplica el LB/gateway (NGINX acá) o se lleva a un contador compartido en Redis (el script Lua de §22).
- **Observabilidad (§8, §14):** las alertas y percentiles deben agregarse **entre** instancias (por eso `histogram_quantile` con `sum by (le)`, y por eso Alertmanager agrupa las alertas de las N réplicas).
- **La base de datos** se convierte en el próximo cuello de botella: réplicas de lectura primero, sharding después (§19/§22-cluster).

**Este proyecto:** la app es stateless a propósito (estado en Mongo/Redis; `server.ts` es una factory sin estado global), así que el `docker-compose` podría levantar `app1..appN` detrás de NGINX sin tocar código — solo agregando upstreams.

**Referencias:**

- [AWS: escalado horizontal vs vertical](https://aws.amazon.com/compare/the-difference-between-horizontal-vertical-scaling/)
- [The Twelve-Factor App: processes (stateless)](https://12factor.net/processes)
- [NGINX: load balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)

---

<a id="sec-28"></a>

## 28. Arquitectura Orientada a Eventos — Caso: Depósito Bancario

La definición del apunte, refinada: en una arquitectura orientada a eventos, **cada acción dentro de un proceso puede emitir un evento** — un hecho inmutable, en pasado ("ocurrió X") — que otros componentes consumen para reaccionar, sin que el emisor sepa quiénes son ni los espere. (Fundamentos y tradeoffs: §17.)

### El caso: procesar un depósito

**Versión sincrónica (acoplada):** el servicio de depósitos llama, en línea, a cada interesado:

```
POST /deposits
  └─► validar → acreditar saldo
      → llamar a notificaciones (email/push)     ← si tarda 2s, el depósito tarda 2s
      → llamar a detección de fraude              ← si está caído, ¿falla el depósito?
      → llamar a contabilidad/ledger
      → llamar a reportes regulatorios
  └─► 200 OK  (después de TODOS)
```

Problemas: la latencia del depósito = suma de todas las llamadas; cada consumidor caído es una decisión de diseño forzada (¿fallo todo o trago el error?); y agregar un interesado nuevo = modificar y redesplegar el servicio de depósitos.

**Versión orientada a eventos:**

```
POST /deposits
  └─► validar → acreditar saldo (transacción ACID local)
  └─► publicar DepositCompleted { depositId, accountId, amount, currency, occurredAt }
  └─► 200 OK   ← el usuario espera SOLO la operación core

              broker (Kafka/SQS/RabbitMQ)
                ├─► notificaciones   → email "recibiste $X"
                ├─► fraude           → scoring async; si es sospechoso, congela después
                ├─► ledger           → asiento contable
                ├─► regulatorio      → reporte
                └─► (futuro) cashback → se suscribe SIN tocar el servicio de depósitos
```

Qué se ganó, en términos de los apuntes y secciones previas:

- **Latencia:** la respuesta al usuario cubre solo la operación esencial — la misma jugada que el fire-and-forget de este proyecto (§7), pero con durabilidad de broker.
- **Desacople / consumidores nuevos gratis:** la ganancia central de §17 — el productor no cambia cuando aparece un consumidor.
- **Absorción de ráfagas y fallas parciales:** notificaciones caídas = eventos que esperan en la cola (con DLQ para los venenosos, §7), no depósitos fallando.
- **Auditoría/replay:** el log de eventos es historia durable de todo lo que pasó — en banca, valor regulatorio directo.

### Los dos detalles que hacen seria la respuesta

1. **Idempotencia en cada consumidor (§22):** el broker entrega at-least-once → el ledger **va a recibir** `DepositCompleted` duplicado alguna vez. Asiento con clave única por `depositId` o chequeo `SET NX` antes de procesar — si no, plata contada dos veces.
2. **Publicación atómica — el patrón outbox:** "acreditar saldo" (DB) y "publicar evento" (broker) son dos sistemas — si el proceso muere entre ambos, tenés saldo sin evento (o evento sin saldo). Solución: escribir el evento en una tabla `outbox` **dentro de la misma transacción** que el saldo, y un relay/CDC lo publica al broker después. Consistencia sin transacción distribuida.

**Consistencia eventual asumida:** el email puede llegar 3 segundos después del depósito — aceptable. Lo que **no** puede ser eventual (validar fondos, acreditar) quedó **dentro** de la transacción sincrónica. Saber trazar esa línea — qué es core sincrónico y qué es reacción asíncrona — es exactamente la profundidad que este caso ejercita en la entrevista.

**Referencias:**

- [Martin Fowler: What do you mean by "Event-Driven"?](https://martinfowler.com/articles/201701-event-driven.html)
- [microservices.io: patrón Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [microservices.io: patrón Saga](https://microservices.io/patterns/data/saga.html)
- [Confluent: Event-Driven Architecture](https://www.confluent.io/learn/event-driven-architecture/)

---

<a id="sec-29"></a>

## 29. Pendientes de Investigación

Del simulacro de entrevista (2026-07-01):

- [ ] **Redis en OB** — estudiar cómo está implementado Redis en el trabajo: ¿cache-aside o write-through? ¿TTLs? ¿cluster o instancia única? ¿se usa para idempotencia/locks o solo cache? Contrastar contra §22.
- [x] **DynamoDB** — investigado → §20. Profundizar después: single-table design en la práctica, DynamoDB Streams + Lambda.
- [x] **OpenSearch** — investigado → §21. Profundizar después: mappings/analyzers, dimensionamiento de shards.
- [ ] **Feedback del simulacro: "falta profundidad en el discurso"** — el patrón de mejora no es saber más términos sino conectarlos: cada respuesta debería incluir (1) definición corta, (2) el tradeoff/costo, (3) un ejemplo concreto propio (este proyecto sirve: §3 circuit breaker, §7 fire-and-forget, §22 idempotencia). Practicar respuestas de 90 segundos con esa estructura.
