# Burp Suite — Análisis de Tráfico HTTP/HTTPS

**Herramienta:** Burp Suite Community Edition v2026.2.3  
**Desarrollador:** PortSwigger Ltd.  
**Entorno:** Kali Linux (x86_64)  
**Estudiante:** Juan Diego Velasco Quintero  
**Fecha:** Tunja, 20 de mayo de 2026  

> Análisis en modo pasivo y observacional sobre el tráfico generado por la propia navegación del estudiante. Sin ataques activos. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción

Burp Suite actúa como proxy de intercepción entre el navegador y el servidor, capturando y permitiendo analizar o modificar el tráfico HTTP/HTTPS en tiempo real. Se trabajaron tres módulos principales.

**Módulos usados:** Proxy · HTTP History · Repeater

---

## Módulos analizados

### Módulo 1 — Proxy / Intercept

El módulo Proxy con `Intercept on` detiene cada petición antes de que llegue al servidor, permitiendo inspeccionar todas sus cabeceras.

**Captura:** Se interceptó una petición GET a `fonts.googleapis.com:443` durante la carga del sitio, con el Inspector lateral mostrando 18 cabeceras HTTP/2.

**Cabeceras capturadas de la petición interceptada:**

| Cabecera | Valor observado |
|---|---|
| `Host` | `fonts.googleapis.com` |
| `User-Agent` | Mozilla/5.0 (X11; Linux x86_64) Chrome/145.0.0.0 |
| `Sec-Ch-Ua-Platform` | `"Linux"` |
| `Accept-Language` | `en-US,en;q=0.9` |
| `Accept` | `text/css,*/*;q=0.1` |
| `Sec-Fetch-Site` | `cross-site` |
| `X-Client-Data` | `CKr8ygE=` |

> ⚠️ **Hallazgo:** El sitio carga fuentes desde `fonts.googleapis.com` con `Sec-Fetch-Site: cross-site` sin Subresource Integrity (SRI) verificable.

---

### Módulo 2 — HTTP History

El panel HTTP History registra todo el tráfico que ha pasado por el proxy, con o sin intercepción activa.

**Peticiones destacadas (filtro: sin CSS ni imágenes):**

| # | Host | Método | Cód. | Observación |
|---|---|---|---|---|
| 2 | `latinoamericacomparte.com` | GET | 200 | Pág. principal; 136,239 bytes HTML |
| 77 | `latinoamericacomparte.com` | GET | 200 | Recarga; mismo tamaño |
| 342 | `colombiacomparte.com` | GET | 200 | Sitio hermano; 279,073 bytes HTML |
| 498 | `colombiacomparte.com` | POST | 200 | `/wp-admin/admin-ajax.php`; JSON |
| 529 | `colombiacomparte.com` | GET | 200 | `/wp-json/sliderrevolution/` |
| 1 | `www.google.com` | GET | 404 | Warmup; recurso no encontrado |

> ⚠️ **Hallazgos críticos:**
> - `colombiacomparte.com` (IP: `192.124.249.102`) es un sitio hermano con WordPress expuesto.
> - `/wp-admin/admin-ajax.php` es un vector frecuente de ataques si no está protegido.
> - El plugin **Slider Revolution** (`/wp-json/sliderrevolution/`) tiene historial de CVEs críticos (RCE en versiones antiguas).

---

### Módulo 3 — Repeater (Solicitud manual)

Permite tomar una solicitud interceptada, modificarla y reenviarla al servidor para observar distintas respuestas.

**Solicitud enviada al Repeater:**

```
GET /css2?family=Poppins:wght@300;400;600;700;800&display=swap HTTP/2
Host: fonts.googleapis.com
Referer: https://latinoamericacomparte.com/
Sec-Fetch-Site: cross-site
Sec-Fetch-Mode: no-cors
Sec-Fetch-Storage-Access: active
Accept-Encoding: gzip, deflate, br
```

La cabecera `Referer` confirma que `latinoamericacomparte.com` realiza una petición cross-site a Google Fonts al cargar cada página. Sin SRI, un atacante podría inyectar CSS malicioso en un escenario de compromiso del CDN.

---

### Módulo 4 — Repeater con Respuesta del Servidor

Se analizó una petición POST a `colombiacomparte.com/wp-admin/admin-ajax.php` con su respuesta HTTP/2 completa.

**Análisis de cabeceras de seguridad en la respuesta:**

| Cabecera | Presente | Valor / Observación |
|---|---|---|
| `X-Xss-Protection` | ✅ | `1; mode=block` |
| `X-Frame-Options` | ✅ | `SAMEORIGIN` |
| `X-Content-Type-Options` | ✅ | `nosniff` |
| `Content-Security-Policy` | ⚠️ | Solo `upgrade-insecure-requests` |
| `Referrer-Policy` | ✅ | `strict-origin-when-cross-origin` |
| `Server` | ❌ | `Sucuri/Cloudproxy` (revelado) |
| `Cache-Control` | ✅ | `no-cache, must-revalidate, no-store` |
| `Expires` | ❌ | `Wed, 11 Jan 1984` (fecha antigua) |

---

## Resumen de hallazgos

| Hallazgo | Nivel | Módulo | Detalle |
|---|---|---|---|
| CDN externo sin SRI | 🟠 Medio | Proxy / Repeater | Google Fonts sin integridad verificable |
| WordPress expuesto | 🟠 Medio | HTTP History | `/wp-admin/admin-ajax.php` accesible |
| Plugin Slider Revolution | 🔴 Alto | HTTP History | Historial de CVEs críticos (RCE) |
| CSP insuficiente | 🟠 Medio | Repeater | Solo `upgrade-insecure-requests` |
| Server banner revelado | 🟠 Medio | Repeater | `Sucuri/Cloudproxy` expuesto |
| Cookies de sesión en POST | 🟠 Medio | Repeater | `sbjs_*` sin cifrado adicional |
| Infraestructura separada | 🟡 Info | HTTP History | IPs distintas para dominios hermanos |

---

## Recomendaciones

1. **Actualizar/proteger Slider Revolution** — verificar la versión del plugin y aplicar parches de seguridad o limitar el acceso al endpoint `/wp-json/sliderrevolution/`.
2. **Reforzar la CSP** — agregar directivas `script-src`, `default-src` y `object-src` para mitigar ataques XSS.
3. **Ocultar la cabecera `Server`** — no revelar el WAF/tecnología utilizada; configurar Sucuri para suprimir o reemplazar este header.
4. **Implementar SRI** en todas las cargas de recursos externos (Google Fonts, Cloudflare CDN).
5. **Revisar cookies de sesión** — asegurarse de que las cookies de seguimiento (`sbjs_*`) tengan los atributos `HttpOnly`, `Secure` y `SameSite=Strict`.
