<!-- ============================================================ -->
<!-- CARÁTULA -->
<!-- ============================================================ -->

<div align="center">

# Universidad Católica del Uruguay

### Desarrollo de Software Seguro

# Práctico 5 - Diseño Seguro - OWASP Threat Modeling Process



**Valentina Blanco**

**Belén Kanas** 



**Profesores:** 

Wiler Alvez

Alejandro Piccardo


**7 de Noviembre de 2026**

----------
</div>


<div style="page-break-after: always;"></div>

<!-- ============================================================ -->
<!-- Índice -->
<!-- ============================================================ -->

<div align="justify">

## Índice

* [Introducción](#introduccion)
* [Diagrama de Arquitectura](#diagrama-de-arquitectura)
* [Modelo de Amenazas](#modelo-de-amenazas)
    * [Paso 1: Determinar el alcance](#paso-1)
    * [Paso 2: Determinar amenazas](#paso-2)
    * [Paso 3: Determinar las contramedidas y mitigaciones](#paso-3)
    * [Paso 4: Evaluar el análisis realizado](#paso-4)
* [Conclusiones](#conclusiones)

----------
</div>

<div style="page-break-after: always;"></div>

<a name="introduccion"></a>

## Introducción

El presente práctico tiene como objetivo aplicar una nueva iteración del [OWASP Threat Modeling Process](https://owasp.org/projects/threat-dragon) sobre un sistema de gestión de torneos que implementa un nuevo tablero de puntajes público en Internet. La idea principal es poder implementar un modelo de amenazas como también realizar un diagrama de arquitectura con los nuevos cambios aplicados.

La federación incorpora un tablero de puntajes público, accesible desde una URL, con resultados en vivo y de torneos anteriores.

**Preocupación del presidente:** que alguien modifique desde Internet la nota de una gimnasta.

**Objetivos de negocio de esta iteración:**

1. Integridad de los puntajes publicados
2. Disponibilidad del tablero durante la competencia
3. Transparencia (principio de equidad deportiva)
4. Protección de datos personales (Ley N.º 18.331)
5. Reputación de la federación

El análisis sigue los cuatro pasos del OWASP Threat Modeling Process: (1) determinar el alcance, (2) determinar amenazas, (3) determinar contramedidas y mitigaciones, y (4) evaluar el análisis realizado.

---

<!-- ============================================================ -->
<!-- Diagrama de Arquitectura -->
<!-- ============================================================ -->

<div style="page-break-after: always;"></div>
<a name="diagrama-de-arquitectura"></a>

## Diagrama de Arquitectura

La arquitectura propuesta se divide en tres zonas separadas por firewalls:

- **Internet:** público anónimo (navegador), que accede por HTTPS.
- **DMZ:** CDN/WAF/reverse proxy, tablero público (SPA estática en nginx), API pública de **solo lectura** (solo GET), BD de publicación (base separada con datos publicables) y monitoreo y logs.
- **Red interna (arena, WiFi con SSID exclusivo):** App Mobile del Juez, Web de Mesa de Control/Admin, API REST interna, BD principal (PostgreSQL) y servicio de sincronización (publicador).

![Figura 1: Arquitectura propuesta para el tablero de puntajes público](diagrama_arquitectura_tablero_publico.png)

*Figura 1: Arquitectura propuesta para el tablero de puntajes público*

### Principios de diseño

1. El tablero público nunca se conecta a la BD principal.
2. No existe ningún endpoint de escritura accesible desde Internet.
3. La sincronización es unidireccional (red interna → DMZ), con mTLS y mensajes firmados.
4. La DMZ no puede iniciar conexiones hacia la red interna.
5. Solo se publican resultados confirmados por la Mesa de Control.
6. La BD de publicación es una base separada, no una réplica de la principal, y contiene solo datos publicables. Tiene dos cuentas: el publicador (única con escritura) y la API pública (solo SELECT).

---

<!-- ============================================================ -->
<!-- Modelo de Amenazas -->
<!-- ============================================================ -->

<div style="page-break-after: always;"></div>
<a name="modelo-de-amenazas"></a>

## Modelo de Amenazas

<a name="paso-1"></a>

### Paso 1: Determinar el alcance

#### Dependencias externas nuevas

| ID | Descripción |
|---|---|
| 6 | Tablero público (SPA) en nginx sobre Linux hardenizado, en la DMZ. Solo contenido estático. |
| 7 | API pública de lectura sobre Linux hardenizado en la DMZ, con usuario de BD de solo lectura. |
| 8 | BD de publicación: base PostgreSQL separada en la DMZ, solo con datos publicables. Cuentas: publicador (escritura) y API pública (solo SELECT). |
| 9 | CDN/WAF/reverse proxy con TLS 1.2 o superior. |
| 10 | Canal de sincronización unidireccional autenticado (mTLS + mensajes firmados). |
| 11 | DNS y certificado TLS de la URL pública. |
| 12 | Firewalls entre Internet, DMZ y red interna. |
| 13 | Monitoreo y logs centralizados de la DMZ. |

#### Puntos de entrada nuevos

| ID | Nombre | Descripción | Nivel de confianza |
|---|---|---|---|
| 4 | Puerto HTTPS público | Único puerto expuesto a Internet (CDN/WAF) | (5) Anónimo de Internet |
| 4.1 | Tablero público estático | Contenido estático de la SPA | (5) |
| 4.2 | API pública: resultados en vivo | GET (SSE o polling) | (5) |
| 4.3 | API pública: históricos | GET de torneos anteriores | (5) |
| 5 | Canal de sincronización | Publicación de resultados confirmados hacia la DMZ | (6) Servicio de sincronización |

#### Niveles de confianza

Se mantienen (1) Anónimo de la red local, (2) Administrador, (3) Mesa de Control y (4) Juez. Se agregan (5) Anónimo de Internet, solo lectura de datos publicados, y (6) Servicio de sincronización, cuenta de servicio sin privilegios de administración.

#### Activos principales

Notas de los gimnastas (integridad, crítico), resultados publicados, datos personales de gimnastas y jueces, credenciales y claves del canal de sincronización, disponibilidad del tablero, aislamiento de la red interna.

#### DFD: flujos nuevos (continúan la numeración del DFD nivel 2 existente)

| Flujo | Origen → Destino | Datos |
| --- | --- | --- |
| 12 | Público → CDN/WAF | Pedidos HTTPS de lectura |
| 13 | CDN/WAF → Tablero estático | Contenido estático (SPA) |
| 14 | CDN/WAF → API pública | GET de resultados en vivo e históricos |
| 15 | API pública → BD de publicación | Consultas SELECT |
| 16 | BD principal → Servicio de sincronización | Resultados confirmados |
| 17 | Servicio de sincronización → BD de publicación | Resultados firmados (mTLS) |

<a name="paso-2"></a>

### Paso 2: Determinar amenazas

Las amenazas se identificaron aplicando **STRIDE** sobre los componentes y flujos nuevos del **DFD** (flujos 12 a 17) y se priorizaron con **DREAD**.

Orden de las métricas:

1. Damage
2. Reproducibility
3. Exploitability
4. Affected users
5. Discoverability

Discoverability se asume en 10 salvo que haya una buena razón, y el puntaje es el promedio de las cinco. Se valora el riesgo inherente, es decir, sin la mitigación propuesta.

Para expresar el puntaje como nivel de severidad se usa la escala cualitativa de **CVSS** v4.0:

- **Crítica - 9.0 a 10.0**
- **Alta - 7.0 a 8.9**
- **Media - 4.0 a 6.9**
- **Baja - 0.1 a 3.9**

Los valores DREAD son una estimación del equipo y se pueden recalibrar.

| ID | STRIDE | Amenaza | Flujo | DREAD (D,R,E,A,D) | Severidad |
| --- | --- | --- | --- | --- | --- |
| T14 | Tampering | Se publica una nota provisoria o con error de tipeo | 16-17 | (8,7,10,10,10) = **9.0** | Crítica |
| T1 | Tampering | Modificar una nota desde Internet (endpoint de escritura o método HTTP no restringido) | 12-15 | (10,8,5,10,10) = **8.6** | Alta |
| T2 | Tampering | Inyección SQL en la API pública | 14-15 | (8,8,6,10,10) = **8.4** | Alta |
| T10 | Denial of Service | DDoS o pico de tráfico durante el torneo | 12-14 | (6,8,8,10,10) = **8.4** | Alta |
| T8 | Information disclosure | La API expone datos no publicables (personales, notas provisorias, identidad de jueces) | 14-15 | (8,8,5,10,10) = **8.2** | Alta |
| T3 | Spoofing / Tampering | Suplantación del servicio de sincronización para publicar notas falsas | 17 | (10,6,4,10,10) = **8.0** | Alta |
| T6 | Spoofing | Sitio falso (phishing, typosquatting) o secuestro de DNS | 12 | (7,8,6,9,10) = **8.0** | Alta |
| T12 | Elevation of privilege | Compromiso de la DMZ y pivoteo a la red interna | 17 | (10,5,5,10,10) = **8.0** | Alta |
| T5 | Tampering | XSS o defacement del tablero | 13 | (6,8,5,10,10) = **7.8** | Alta |
| T13 | Elevation of privilege | Explotación de software o librerías vulnerables | 13-15 | (7,6,6,10,10) = **7.8** | Alta |
| T4 | Tampering | Replay o alteración de mensajes de sincronización | 17 | (7,5,3,10,10) = **7.0** | Alta |
| T7 | Tampering / Information disclosure | Interceptación del tráfico por falta de TLS | 12-14 | (3,10,5,5,10) = **6.6** | Media |
| T11 | Denial of Service | Scraping masivo o conexiones SSE persistentes | 14 | (2,10,8,3,10) = **6.6** | Media |
| T17 | Denial of Service | Caída o bloqueo del servicio de sincronización: los resultados en vivo dejan de actualizarse | 16-17 | (4,5,4,10,10) = **6.6** | Media |
| T15 | Repudiation | Cambios o publicaciones sin trazabilidad | 16-17 | (6,10,3,5,5) = **5.8** | Media |
| T9 | Information disclosure | Errores detallados o banners de versión | 13-14 | (1,10,5,2,10) = **5.6** | Media |
| T16 | Information disclosure | Lectura de los mensajes de sincronización en tránsito | 17 | (5,5,3,5,10) = **5.6** | Media |

<a name="paso-3"></a>

### Paso 3: Determinar las contramedidas y mitigaciones

Para cada amenaza se eligió un tratamiento (aceptar, eliminar, mitigar o transferir) y se estimó el riesgo residual.

| ID | Tratamiento | Mitigación | Riesgo residual |
| --- | --- | --- | --- |
| T14 | Mitigar | Solo se publican resultados confirmados por la Mesa de Control; las correcciones requieren doble control y quedan versionadas y visibles | Medio |
| T1 | **Eliminar** | No existe endpoint de escritura en la DMZ; solo GET/HEAD y el WAF bloquea el resto; la API usa una cuenta de BD de solo SELECT; no hay ruta hacia la BD principal | Bajo |
| T2 | Mitigar | Consultas parametrizadas, validación con lista blanca, cuenta de solo lectura, WAF | Bajo |
| T10 | Mitigar y Transferir | CDN con caché de 1 a 2 s, protección anti-DDoS del proveedor, rate limiting, escalado; el sistema interno sigue operando si cae el tablero | Medio |
| T8 | Mitigar | BD de publicación con solo datos publicables, DTOs explícitos, revisión de privacidad (Ley 18.331) | Bajo |
| T3 | Mitigar | mTLS, mensajes firmados, gestor de secretos con rotación de claves, firewall por IP y puerto | Bajo |
| T6 | Mitigar | Dominio oficial, HSTS, DNSSEC si es posible, difusión de la URL oficial por canales de la federación | Medio |
| T12 | Mitigar | Segmentación con firewall (la DMZ no inicia conexiones hacia adentro), sin credenciales internas en la DMZ, mínimo privilegio, hardening | Bajo |
| T5 | Mitigar | Escape de salida, CSP estricta, cabeceras de seguridad, SPA sin contenido ingresado por usuarios | Bajo |
| T13 | Mitigar | Parches, actualización de dependencias, análisis SCA en el pipeline | Bajo |
| T4 | Mitigar | Identificador y marca de tiempo por mensaje, firma y verificación de integridad | Bajo |
| T7 | Mitigar | TLS 1.2+ obligatorio, redirección HTTP a HTTPS, HSTS, certificado de CA reconocida | Bajo |
| T11 | **Aceptar** (parcial) | Los resultados son públicos por el principio de transparencia; se acepta el scraping razonable, con rate limiting y límite de conexiones por cliente | Bajo |
| T17 | Mitigar | Monitoreo y alertas sobre el publicador, reintentos con cola de mensajes, indicador de "última actualización" en el tablero y publicación manual de contingencia por la Mesa de Control; el sistema interno sigue operando | Bajo |
| T15 | Mitigar | Auditoría (usuario, fecha, valor anterior y nuevo, hash de lo publicado), logs centralizados y alertas | Bajo |
| T9 | Mitigar | Errores genéricos al cliente, detalle solo en logs internos | Bajo |
| T16 | Mitigar | mTLS con TLS 1.2+ en el canal, y mensajes con solo datos publicables (sin datos personales ni notas provisorias) | Bajo |

La aceptación de T11 la debe formalizar el dueño del proceso de negocio (la federación), no el equipo técnico.

<a name="paso-4"></a>

### Paso 4: Evaluar el análisis realizado

Tras aplicar las mitigaciones, ninguna amenaza queda con riesgo residual alto. Las que mantienen riesgo residual medio son:

- **T14 (nota errónea):** depende de un proceso humano. La mitigación técnica (confirmación y control cuatro ojos) reduce el riesgo, pero no lo elimina.
- **T10 (DDoS):** depende de la capacidad del proveedor de CDN; el riesgo se transfiere en buena parte.
- **T6 (sitio falso):** la federación no controla del todo a los usuarios finales.

Pendientes para una próxima iteración:

- Definir los umbrales de rate limiting y de conexiones SSE
- La política de visibilidad de correcciones de notas
- Elegir proveedor de CDN y su SLA
- Validar con la federación qué datos de gimnastas se publican (si hay menores, evaluar la Ley 18.331)
- Definir monitoreo y respuesta a incidentes de la DMZ

---

<div style="page-break-after: always;"></div>

<a name="conclusiones"></a>

## Conclusiones

A lo largo de este práctico aplicamos una nueva iteración del **OWASP Threat Modeling Process** sobre el tablero de puntajes público, desde la descomposición de la arquitectura hasta la evaluación del riesgo residual.

La arquitectura hace inviable modificar una nota desde Internet: el tablero público es de solo lectura, no tiene endpoints de escritura, no accede a la BD principal y solo recibe datos ya confirmados por un canal unidireccional autenticado. Así se atiende la preocupación del presidente de la federación sin comprometer la disponibilidad ni la confidencialidad de los datos.


---

> Valentina Blanco & Belén Kanas | Desarrollo de Software Seguro 2026
