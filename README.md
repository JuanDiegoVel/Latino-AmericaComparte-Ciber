# Proyecto de Seguridad Informática — latinoamericacomparte.com

**Universidad Santo Tomás, Seccional Tunja**  
**Facultad de Ingeniería de Sistemas**  
**Estudiante:** Juan Diego Velasco Quintero  
**Asignatura:** Seguridad Informática / Ciberseguridad  
**Fecha:** Tunja, 20 de mayo de 2026  
**Entorno:** Kali Linux  

> Todos los análisis fueron realizados con fines exclusivamente académicos, en entorno controlado y autorizado. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción general

Este repositorio reúne cinco análisis de seguridad realizados sobre el sitio web **`https://latinoamericacomparte.com`**, abarcando desde reconocimiento pasivo hasta análisis de tráfico HTTP, clonación del sitio y análisis de exposición de código fuente. Cada análisis usa una herramienta distinta y documenta hallazgos, capturas de terminal y recomendaciones desde la perspectiva Blue Team.

---

## Estructura del proyecto

```
.
├── README.md                          ← Este archivo (visión general)
├── DNS_Footprinting/
│   └── README.md                      ← nslookup · dig · whois
├── Maltego/
│   └── README.md                      ← Reconocimiento OSINT e infraestructura
├── OWASP_ZAP/
│   └── README.md                      ← Análisis pasivo de seguridad web
├── Burp_Suite/
│   └── README.md                      ← Análisis de tráfico HTTP/HTTPS
└── Clonacion_HTTrack/
    ├── README.md                      ← Clonación del sitio y análisis de exposición
    ├── ClonacionSitio_LatinoamericaComparte.tex
    ├── Clonacion_1.png  →  Salida verbose de HTTrack
    ├── Clonacion_2.png  →  Servidor HTTP local (logs 200 OK)
    ├── Clonacion_3.png  →  grep "api"
    ├── Clonacion_4.png  →  grep "fetch"
    ├── Clonacion_5.png  →  grep "token" / "key"
    └── Clonacion_6.png  →  grep "http"
```

---

## Resumen de cada análisis

### 1. DNS Footprinting Pasivo — `nslookup · dig · whois`

Se realizó reconocimiento DNS completamente pasivo usando las herramientas de línea de comandos de Kali Linux. Se consultaron registros A, NS, PTR, DNSKEY y DS, además del registro WHOIS del dominio.

**Hallazgos principales:**
- IP del servidor: `64.202.187.143`
- Nameservers GoDaddy: `ns05/ns06.domaincontrol.com`
- **DNSSEC no implementado** → vulnerable a DNS Cache Poisoning
- TTL atípicamente bajo (159 s) → posible riesgo de DNS rebinding
- Dominio joven (creado agosto 2025, vence agosto 2026)
- Shared hosting detectado: el PTR de una IP vecina resuelve a `copilots.dev`
- Contacto admin DNS expuesto: `dns@jomax.net` (GoDaddy)

---

### 2. Maltego — Reconocimiento OSINT e Infraestructura

Se utilizó Maltego CE para construir un grafo de infraestructura completo del dominio objetivo mediante transforms sobre fuentes públicas (DNS, WHOIS, Wayback Machine, redes sociales). No se generó tráfico directo hacia el servidor.

**Hallazgos principales:**
- Más de 15 subdominios descubiertos, incluyendo `cpanel.`, `whm.`, `webdisk.`, `aulavirtual.`, `www.admin.`
- Panel administrativo `www.admin.latinoamericacomparte.com` visible en DNS público
- Todos los servicios críticos comparten la misma IP (`64.202.187.143`) en shared hosting `/24` (AS398101)
- Capturas de Wayback Machine (feb–mar 2026), incluyendo versiones HTTP sin TLS
- Presencia confirmada en Instagram
- Email del registrador `dns@jomax.net` visible en el grafo

---

### 3. OWASP ZAP — Análisis Pasivo de Seguridad Web

Se configuró Firefox para enrutar el tráfico a través del proxy ZAP (`localhost:8080`) y se navegó manualmente el sitio. Se usaron los módulos History, Spider, AJAX Spider y Client Spider, todos en modo pasivo observacional. No se ejecutó Active Scan sobre el servidor de producción.

**Hallazgos principales:**
- Plataforma Moodle (`/aulavirtual/`) identificada con rutas `/lib`, `/theme`, `pluginfile.php`
- Alertas de nivel **Medium** en sitio principal y aula virtual: `Form, Hidden, Script`, `Password`, `SetCookie`
- AJAX Spider rastreó 68 URLs; imagenes sin control de acceso con tamaños hasta 7.1 MB
- Client Spider descubrió rutas autenticadas: `/aulavirtual/login/`, `/aulavirtual/my/`
- CDNs externos (Google Fonts, Cloudflare) sin verificación de Subresource Integrity (SRI)

---

### 4. Burp Suite — Análisis de Tráfico HTTP/HTTPS

Se usó Burp Suite Community Edition como proxy de intercepción para capturar y analizar el tráfico generado al navegar el sitio. Se trabajaron los módulos Proxy/Intercept, HTTP History y Repeater.

**Hallazgos principales:**
- Sitio hermano `colombiacomparte.com` descubierto en el historial (IP distinta: `192.124.249.102`)
- Endpoint WordPress expuesto: `/wp-admin/admin-ajax.php` y `/wp-json/sliderrevolution/` (historial de CVEs críticos)
- Carga de Google Fonts con `Sec-Fetch-Site: cross-site` sin SRI verificable
- CSP solo contiene `upgrade-insecure-requests` → prácticamente ineficaz contra XSS
- Cabecera `Server: Sucuri/Cloudproxy` revela el WAF utilizado
- Cookies de sesión (`sbjs_*`) enviadas en POST sin cifrado adicional

---

### 5. Clonación de Sitio y Análisis de Exposición — HTTrack

Se clonó el sitio completo con HTTrack Website Copier, se levantó un servidor HTTP local y se analizó el código fuente del espejo con búsquedas `grep` para detectar exposición de APIs, tokens, claves y URLs externas.

**Hallazgos principales:**
- 76 links escaneados, 66 archivos descargados (≈ 38.8 MB) en 1 min 26 s
- 9 errores 404 en rutas referenciadas (`/quienes.html`, `/paises.html`, etc.) → navegación SPA sin páginas independientes
- Único servicio externo real: `api.web3forms.com/submit` para el formulario de contacto
- No se encontraron tokens, claves API ni secretos expuestos en el frontend
- Las búsquedas de `token` y `key` solo retornaron código de la librería `anime.min.js` (tercero)
- Buenas prácticas básicas de seguridad frontend confirmadas

---

## Herramientas utilizadas

| Herramienta | Versión | Módulo / Uso |
|---|---|---|
| nslookup / dig / whois | GNU / BIND 9.20 | DNS Footprinting pasivo |
| Maltego CE | Kali Linux 2026.1 | OSINT, grafos de infraestructura |
| OWASP ZAP | 2.17.0 | Proxy pasivo, Spider, AJAX Spider |
| Burp Suite Community | v2026.2.3 | Proxy, HTTP History, Repeater |
| HTTrack Website Copier | 3.49-6 | Clonación de sitio, análisis grep |

---

## Metodología general

Todos los análisis siguen un enfoque de **reconocimiento pasivo y observacional**:

1. No se ejecutaron ataques activos ni exploits sobre el servidor de producción.
2. Se usó exclusivamente tráfico generado por la navegación propia del estudiante o consultas a fuentes DNS/WHOIS públicas.
3. Los hallazgos se documentan con fines defensivos (perspectiva Blue Team).
4. Cumplimiento estricto de la **Ley 1273 de 2009** — Delitos Informáticos (Colombia).

---

## Recomendaciones consolidadas

| Prioridad | Recomendación | Herramienta que lo detectó |
|---|---|---|
| 🔴 Alta | Implementar DNSSEC | dig / whois |
| 🔴 Alta | Ocultar subdominios administrativos (cpanel, whm, www.admin) | Maltego |
| 🟠 Media | Reforzar CSP (agregar `script-src`, `default-src`) | Burp Suite |
| 🟠 Media | Implementar SRI en recursos CDN externos | OWASP ZAP / Burp Suite |
| 🟠 Media | Normalizar TTL DNS (de 159 s a 3600–86400 s) | dig |
| 🟠 Media | Ocultar cabecera `Server` (no revelar WAF/tecnología) | Burp Suite |
| 🟡 Baja | Resolver rutas 404 referenciadas en el HTML | HTTrack |
| 🟡 Baja | Renovar dominio antes de agosto 2026 | whois |
| 🟡 Baja | Optimizar imágenes (hasta 7.1 MB detectados) | OWASP ZAP |
