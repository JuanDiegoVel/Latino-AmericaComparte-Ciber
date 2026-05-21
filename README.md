# Proyecto de Seguridad Informática — latinoamericacomparte.com

**Universidad Santo Tomás, Seccional Tunja**  
**Facultad de Ingeniería de Sistemas**  
**Estudiante:** Juan Diego Velasco Quintero  
**Asignatura:** Seguridad Informática / Ciberseguridad  
**Fecha:** Tunja, 20 de mayo de 2026  
**Entorno:** Kali Linux  

> Todos los análisis fueron realizados con fines exclusivamente académicos, en entorno controlado y autorizado. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## ¿Cómo está organizado este repositorio?

Cada laboratorio vive en su **propia rama**. Dentro de cada rama encontrarás el documento PDF del laboratorio y su README explicativo.

| Rama | Documento | README |
|---|---|---|
| `main` | *(este archivo)* | README general del proyecto |
| `dns-footprinting` | `DNS_ProyectoCiber.pdf` | `README_DNS_Footprinting.md` |
| `maltego-osint` | `Maltego_ProyectoCiber.pdf` | `README_Maltego.md` |
| `owasp-zap` | `OWAS_ProyectoCiber.pdf` | `README_OWASP_ZAP.md` |
| `burp-suite` | `BURP_ProyectoCiber.pdf` | `README_Burp_Suite.md` |
| `clonacion-httrack` | `Clonacion_ProyectoCiber.pdf` + imágenes + `.tex` | `README_Clonacion_HTTrack.md` |

> 💡 Para ver el contenido de cada laboratorio cambia de rama usando el menú desplegable **"main ▾"** arriba a la izquierda en GitHub.

---

## Descripción general

Este repositorio reúne cinco análisis de seguridad realizados sobre el sitio web **`https://latinoamericacomparte.com`**, abarcando desde reconocimiento pasivo hasta análisis de tráfico HTTP, clonación del sitio y análisis de exposición de código fuente. Cada análisis usa una herramienta distinta y documenta hallazgos, capturas de terminal y recomendaciones desde la perspectiva Blue Team.

---

## Rama 1 — `dns-footprinting`
### DNS Footprinting Pasivo · `nslookup · dig · whois`

📄 **Documento:** `DNS_ProyectoCiber.pdf`  
📋 **README:** `README_DNS_Footprinting.md`

Se realizó reconocimiento DNS completamente pasivo usando herramientas de línea de comandos. Se consultaron registros A, NS, PTR, DNSKEY y DS, además del registro WHOIS del dominio.

**Hallazgos principales:**
- IP del servidor: `64.202.187.143`
- Nameservers GoDaddy: `ns05/ns06.domaincontrol.com`
- **DNSSEC no implementado** → vulnerable a DNS Cache Poisoning
- TTL atípicamente bajo (159 s) → posible riesgo de DNS rebinding
- Dominio joven (creado agosto 2025, vence agosto 2026)
- Shared hosting: PTR de IP vecina resuelve a `copilots.dev`

---

## Rama 2 — `maltego-osint`
### Maltego · Reconocimiento OSINT e Infraestructura

📄 **Documento:** `Maltego_ProyectoCiber.pdf`  
📋 **README:** `README_Maltego.md`

Se usó Maltego CE para construir un grafo de infraestructura completo mediante transforms sobre fuentes públicas (DNS, WHOIS, Wayback Machine). Sin tráfico directo al servidor.

**Hallazgos principales:**
- Más de 15 subdominios descubiertos: `cpanel.`, `whm.`, `webdisk.`, `aulavirtual.`, `www.admin.`
- Panel administrativo `www.admin.latinoamericacomparte.com` visible en DNS público
- Todos los servicios críticos comparten la misma IP en shared hosting `/24` (AS398101)
- Capturas de Wayback Machine (feb–mar 2026) incluyendo versiones HTTP sin TLS

---

## Rama 3 — `owasp-zap`
### OWASP ZAP · Análisis Pasivo de Seguridad Web

📄 **Documento:** `OWAS_ProyectoCiber.pdf`  
📋 **README:** `README_OWASP_ZAP.md`

Se configuró Firefox para enrutar el tráfico a través del proxy ZAP (`localhost:8080`). Se usaron los módulos History, Spider, AJAX Spider y Client Spider en modo pasivo. Sin Active Scan.

**Hallazgos principales:**
- Plataforma Moodle identificada con rutas `/lib`, `/theme`, `pluginfile.php`
- Alertas **Medium**: `Form, Hidden, Script`, `Password`, `SetCookie`
- AJAX Spider rastreó 68 URLs; imágenes sin control de acceso hasta 7.1 MB
- Client Spider descubrió rutas autenticadas: `/aulavirtual/login/`, `/aulavirtual/my/`

---

## Rama 4 — `burp-suite`
### Burp Suite · Análisis de Tráfico HTTP/HTTPS

📄 **Documento:** `BURP_ProyectoCiber.pdf`  
📋 **README:** `README_Burp_Suite.md`

Se usó Burp Suite Community Edition como proxy de intercepción. Módulos trabajados: Proxy/Intercept, HTTP History y Repeater.

**Hallazgos principales:**
- Sitio hermano `colombiacomparte.com` descubierto (IP: `192.124.249.102`)
- WordPress expuesto: `/wp-admin/admin-ajax.php` y `/wp-json/sliderrevolution/` (CVEs críticos)
- CSP solo contiene `upgrade-insecure-requests` → ineficaz contra XSS
- Cabecera `Server: Sucuri/Cloudproxy` revela el WAF utilizado

---

## Rama 5 — `clonacion-httrack`
### HTTrack · Clonación de Sitio y Análisis de Exposición

📄 **Documento:** `Clonacion_ProyectoCiber.pdf`  
📄 **LaTeX:** `ClonacionSitio_LatinoamericaComparte.tex`  
🖼️ **Capturas:** `Clonacion_1.png` al `Clonacion_6.png`  
📋 **README:** `README_Clonacion_HTTrack.md`

Se clonó el sitio completo con HTTrack, se levantó un servidor HTTP local y se analizó el código fuente del espejo con `grep` para detectar APIs, tokens, claves y URLs externas.

**Hallazgos principales:**
- 76 links escaneados, 66 archivos descargados (≈ 38.8 MB) en 1 min 26 s
- 9 errores 404 en rutas referenciadas → navegación SPA sin páginas independientes
- No se encontraron tokens, claves API ni secretos expuestos en el frontend
- Búsquedas de `token` y `key` solo retornaron código de `anime.min.js` (tercero)

---

## Herramientas utilizadas

| Herramienta | Versión | Rama |
|---|---|---|
| nslookup / dig / whois | GNU / BIND 9.20 | `dns-footprinting` |
| Maltego CE | Kali Linux 2026.1 | `maltego-osint` |
| OWASP ZAP | 2.17.0 | `owasp-zap` |
| Burp Suite Community | v2026.2.3 | `burp-suite` |
| HTTrack Website Copier | 3.49-6 | `clonacion-httrack` |

---

## Metodología general

Todos los análisis siguen un enfoque de **reconocimiento pasivo y observacional**:

1. No se ejecutaron ataques activos ni exploits sobre el servidor de producción.
2. Se usó exclusivamente tráfico generado por la navegación propia del estudiante o consultas a fuentes DNS/WHOIS públicas.
3. Los hallazgos se documentan con fines defensivos (perspectiva Blue Team).
4. Cumplimiento estricto de la **Ley 1273 de 2009** — Delitos Informáticos (Colombia).

---

## Recomendaciones consolidadas

| Prioridad | Recomendación | Rama donde se detectó |
|---|---|---|
| 🔴 Alta | Implementar DNSSEC | `dns-footprinting` |
| 🔴 Alta | Ocultar subdominios administrativos (cpanel, whm, www.admin) | `maltego-osint` |
| 🟠 Media | Reforzar CSP (`script-src`, `default-src`) | `burp-suite` |
| 🟠 Media | Implementar SRI en recursos CDN externos | `owasp-zap` / `burp-suite` |
| 🟠 Media | Normalizar TTL DNS (159 s → 3600–86400 s) | `dns-footprinting` |
| 🟠 Media | Ocultar cabecera `Server` (no revelar WAF) | `burp-suite` |
| 🟡 Baja | Resolver rutas 404 referenciadas en el HTML | `clonacion-httrack` |
| 🟡 Baja | Renovar dominio antes de agosto 2026 | `dns-footprinting` |
| 🟡 Baja | Optimizar imágenes (hasta 7.1 MB detectados) | `owasp-zap` |
