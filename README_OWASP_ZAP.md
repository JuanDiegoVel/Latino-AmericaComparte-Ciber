# OWASP ZAP — Análisis Pasivo de Seguridad Web

**Herramienta:** OWASP ZAP 2.17.0 (Zed Attack Proxy)  
**Entorno:** Kali Linux — Firefox proxiado a `localhost:8080`  
**Estudiante:** Juan Diego Velasco Quintero  
**Fecha:** Tunja, 20 de mayo de 2026  

> Análisis en modo pasivo y observacional. No se ejecutó Active Scan sobre el servidor de producción. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción

OWASP ZAP actúa como proxy de intercepción entre el navegador y el servidor, capturando todo el tráfico HTTP/HTTPS y analizándolo de forma pasiva para detectar alertas de seguridad. Se trabajaron cuatro módulos en modo de solo lectura.

**Datos del objetivo:**

| Campo | Valor |
|---|---|
| URL principal | `https://latinoamericacomparte.com/` |
| Aula virtual | `https://www.latinoamericacomparte.com/aulavirtual/` |
| CDN detectado | Cloudflare (`cdnjs.cloudflare.com`) |
| Fuentes web | Google Fonts (`fonts.googleapis.com`, `fonts.gstatic.com`) |
| Tecnología AV | Moodle (detectado por rutas `/lib`, `/theme`, `pluginfile.php`) |

---

## Módulos analizados

### Módulo 1 — History (Historial de Tráfico)

El módulo History registra pasivamente todo el tráfico HTTP/HTTPS interceptado: método, URL, código de respuesta, tamaño del cuerpo, RTT y alertas.

**Peticiones representativas:**

| ID | URL | Código | Observación |
|---|---|---|---|
| 1 | `https://latinoamericacomparte.com/` | 200 | Alertas Medium |
| 23 | `/aulavirtual/` (GET) | 200 | Aula Moodle accesible |
| 143 | `/aulavirtual/` (POST) | 200 | Envío de formulario; 4,930 bytes |
| 92 | `/aulavirtual/` (GET) | 200 | 62,976 bytes — alertas Password/Script/SetCookie |
| 3 | `fonts.googleapis.com/css2` | 200 | Comment detectado |
| 5 | `cdnjs.cloudflare.com/animejs/3.2.1` | 200 | Librería JS externa |

> ⚠️ **Hallazgo:** La petición con ID 92 hacia `/aulavirtual/` recibe 62,976 bytes y está marcada con alertas Medium relacionadas con `Password`, `Script` y `SetCookie`.

---

### Módulo 2 — Spider / Mapa de Rutas

El Spider sigue automáticamente todos los enlaces HTML para construir un árbol completo de la estructura del sitio.

**Rutas y dominios descubiertos:**

| Dominio / Ruta | Observación |
|---|---|
| `/aulavirtual/` | Plataforma Moodle |
| `/aulavirtual/lib/` | Librerías del core de Moodle |
| `/aulavirtual/pluginfile.php` | Entrega de archivos Moodle (vector conocido) |
| `/aulavirtual/theme/` | Tema visual de Moodle |
| `/img/COACHES/`, `/img/MENTORES/` | Imágenes sin control de acceso |
| `https://fonts.gstatic.com` | CDN de fuentes Google |
| `https://cdnjs.cloudflare.com/ajax/` | CDN de librerías JS |

> ⚠️ **Hallazgo:** La ruta `pluginfile.php` es un vector conocido en instalaciones Moodle sin configuración adecuada de control de acceso.

---

### Módulo 3 — AJAX Spider

Utiliza un navegador real en modo headless para ejecutar JavaScript y descubrir rutas dinámicas que el Spider clásico no detecta.

**Estadísticas:**

| Parámetro | Valor |
|---|---|
| URLs rastreadas | 68 |
| Peticiones in-scope (200 OK) | ≈ 60 |
| Peticiones out-of-scope (403) | 4 (Mozilla CDN — no relevantes) |
| Alertas detectadas | Medium (sitio principal), Informational (CDN) |

> ⚠️ **Hallazgo:** Alerta Medium relacionada con `Form, Hidden, Script` — posibles campos de formulario no visibles que procesan datos sensibles.

---

### Módulo 4 — Client Spider (Reconocimiento Profundo)

Extiende el rastreo simulando interacción real con la interfaz: clicks, navegación entre secciones autenticadas y formularios de login.

**Rutas autenticadas descubiertas:**

| Ruta | Relevancia |
|---|---|
| `/aulavirtual/login/` | Punto de autenticación Moodle |
| `/aulavirtual/my/` | Dashboard del usuario autenticado |
| `/aulavirtual/lib/` | Librerías del core (acceso directo) |
| `/aulavirtual/theme/` | Tema Moodle (posibles archivos PHP) |

**Estadísticas al momento de captura:** 8% de progreso · 194 URLs rastreadas · 14 nodos añadidos.

---

## Resumen de hallazgos

| Hallazgo | Nivel | Módulo | Detalle |
|---|---|---|---|
| Alerta sitio principal | 🟠 Medium | History / AJAX Spider | `Form, Hidden, Script` detectados |
| Alerta aulavirtual (ID 92) | 🟠 Medium | History | `Password`, `Script`, `SetCookie` |
| Alerta Google Fonts | 🟠 Medium | History | Comentario HTML detectado |
| Directorios imágenes expuestos | 🟠 Medium | Spider / AJAX | Sin control de acceso visible |
| Rutas Moodle sensibles | 🟠 Medium | Client Spider | `/login/`, `/my/`, `pluginfile.php` |
| CDN sin SRI | 🟠 Medium | History | Google Fonts sin integridad verificable |

---

## Recomendaciones

1. **Implementar SRI** (`Subresource Integrity`) en todas las etiquetas `<link>` y `<script>` que carguen recursos de CDN externos.
2. **Revisar cabeceras de seguridad del aula virtual**: `HttpOnly`, `SameSite=Strict`, `Secure` en las cookies de sesión.
3. **Restringir acceso a directorios de imágenes** — deshabilitar el listado de directorios en el servidor.
4. **Proteger `pluginfile.php`** con autenticación adecuada para evitar acceso anónimo a archivos Moodle.
