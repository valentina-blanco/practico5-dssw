# Práctico 5: Diseño seguro — Tablero de puntajes público

**Desarrollo de Software Seguro (UCU)** · Nueva iteración del OWASP Threat Modeling Process

## 1. Arquitectura propuesta

El diagrama completo está en `diagrama_arquitectura_tablero_publico.drawio`. La arquitectura se divide en tres zonas separadas por firewalls:

- **Internet:** público anónimo (navegador), que accede por HTTPS.
- **DMZ:** CDN/WAF/reverse proxy, tablero público (SPA estática en nginx), API pública de **solo lectura** (solo GET) y BD de publicación (réplica de solo lectura con datos publicables).
- **Red interna (arena, WiFi con SSID exclusivo):** App Mobile del Juez, Web de Mesa de Control/Admin, API REST interna, BD principal (PostgreSQL) y servicio de sincronización.

**Principios de diseño**
1. El tablero público nunca se conecta a la BD principal.
2. No existe ningún endpoint de escritura accesible desde Internet.
3. La sincronización es unidireccional (red interna → DMZ), con mTLS y mensajes firmados.
4. La DMZ no puede iniciar conexiones hacia la red interna.
5. Solo se publican resultados confirmados por la Mesa de Control.

## 2. Descomposición (cambios respecto de la versión anterior)

**Dependencias externas nuevas**

| ID | Descripción |
|---|---|
| 6 | Tablero público (SPA) en nginx sobre Linux hardenizado, en la DMZ. Solo contenido estático. |
| 7 | API pública de lectura sobre Linux hardenizado en la DMZ, con usuario de BD de solo lectura. |
| 8 | BD de publicación: réplica PostgreSQL de solo lectura en la DMZ. |
| 9 | CDN/WAF/reverse proxy con TLS 1.2 o superior. |
| 10 | Canal de sincronización unidireccional autenticado (mTLS + mensajes firmados). |
| 11 | DNS y certificado TLS de la URL pública. |
| 12 | Firewalls entre Internet, DMZ y red interna. |

**Puntos de entrada nuevos**

| ID | Nombre | Descripción | Nivel de confianza |
|---|---|---|---|
| 4 | Puerto HTTPS público | Único puerto expuesto a Internet (CDN/WAF) | (5) Anónimo de Internet |
| 4.1 | Tablero público estático | Contenido estático de la SPA | (5) |
| 4.2 | API pública: resultados en vivo | GET (SSE o polling) | (5) |
| 4.3 | API pública: históricos | GET de torneos anteriores | (5) |
| 5 | Canal de sincronización | Publicación de resultados confirmados hacia la DMZ | (6) Servicio de sincronización |

**Niveles de confianza nuevos:** (5) Anónimo de Internet, solo lectura de datos publicados; (6) Servicio de sincronización, cuenta de servicio sin privilegios de administración. Se mantienen Anónimo (red local), Administrador, Mesa de Control y Juez.

**Activos principales:** notas de los gimnastas (integridad, crítico), resultados publicados, datos personales, credenciales y claves del canal de sincronización, disponibilidad del tablero.

## 3. Amenazas (STRIDE) y mitigaciones

| ID | STRIDE | Amenaza | Riesgo | Mitigación |
|---|---|---|---|---|
| T1 | Tampering | Modificar una nota desde Internet por un endpoint de escritura o método HTTP no restringido | **Alto** | Solo GET/HEAD en la API pública; el WAF bloquea el resto; no hay endpoints de escritura en la DMZ. BD de publicación accedida con usuario de solo lectura. |
| T2 | Tampering | Inyección SQL en la API pública | **Alto** | Consultas parametrizadas, validación con lista blanca, usuario de BD de solo lectura. |
| T3 | Spoofing / Tampering | Suplantación del servicio de sincronización para publicar notas falsas | **Alto** | mTLS, mensajes firmados, gestor de secretos y rotación de claves, firewall por IP y puerto. |
| T4 | Tampering | Replay o alteración de mensajes de sincronización | Medio | Identificador y marca de tiempo en cada mensaje, firma y verificación de integridad. |
| T5 | Tampering | XSS o defacement del tablero | Medio | Escape de salida, CSP estricta, cabeceras de seguridad, SPA sin contenido ingresado por usuarios. |
| T6 | Spoofing | Sitio falso (phishing, typosquatting) o secuestro de DNS | Medio | Dominio oficial, HSTS, DNSSEC si es posible, difusión de la URL oficial. |
| T7 | Tampering / Info. disclosure | Interceptación del tráfico por falta de TLS | Medio | TLS 1.2+ obligatorio, redirección HTTP→HTTPS, HSTS, CA reconocida. |
| T8 | Information disclosure | La API expone datos no publicables (personales, notas provisorias) | **Alto** | BD de publicación solo con datos publicables; DTOs explícitos en la API. |
| T9 | Information disclosure | Errores detallados o banners de versión | Bajo | Errores genéricos al cliente, detalle solo en logs internos. |
| T10 | Denial of Service | DDoS o pico de tráfico durante el torneo | **Alto** | CDN con caché, protección anti-DDoS, rate limiting, escalado. |
| T11 | Denial of Service | Scraping masivo o conexiones SSE persistentes | Medio | Rate limiting por IP, límite de conexiones por cliente. |
| T12 | Elevation of privilege | Compromiso de la DMZ y pivoteo a la red interna | **Alto** | Segmentación con firewall (la DMZ no inicia conexiones hacia adentro), mínimo privilegio, hardening. |
| T13 | Elevation of privilege | Explotación de software o librerías vulnerables | Medio | Parches, actualización de dependencias, análisis SCA en el pipeline. |
| T14 | Tampering | Publicación de una nota provisoria o errónea | Medio | Solo se publican resultados confirmados por la Mesa de Control; correcciones con control de cuatro ojos. |
| T15 | Repudiation | Cambios en notas sin trazabilidad | Medio | Auditoría (usuario, fecha, valor anterior y nuevo), logs centralizados y alertas. |

## 4. Conclusión

La arquitectura hace inviable modificar una nota desde Internet: el tablero público es de solo lectura, no tiene endpoints de escritura, no accede a la BD principal y solo recibe datos ya confirmados por un canal unidireccional autenticado. Así se atiende la preocupación del presidente de la federación sin comprometer la disponibilidad ni la confidencialidad de los datos.
