# Clonación de Sitio y Análisis de Exposición — HTTrack

**Herramienta:** HTTrack Website Copier 3.49-6  
**Entorno:** Kali Linux  
**Estudiante:** Juan Diego Velasco Quintero  
**Fecha:** Tunja, 20 de mayo de 2026  

> Análisis de exposición técnica en modo White Hat sobre el espejo local del sitio. Sin modificación ni interacción activa con el servidor. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción

Se clonó el sitio `https://latinoamericacomparte.com` usando HTTrack para crear un espejo local completo. Luego se levantó un servidor HTTP local para visualizarlo y se realizaron búsquedas `grep` sobre el código fuente del espejo para detectar posibles exposiciones técnicas (APIs, tokens, claves, URLs externas).

---

## Archivos de este laboratorio

| Archivo | Descripción |
|---|---|
| `ClonacionSitio_LatinoamericaComparte.tex` | Documento LaTeX completo del laboratorio |
| `Clonacion_1.png` | Salida verbose de HTTrack en terminal |
| `Clonacion_2.png` | Logs del servidor HTTP local (200 OK) |
| `Clonacion_3.png` | Resultado de `grep -r "api" .` |
| `Clonacion_4.png` | Resultado de `grep -r "fetch" .` |
| `Clonacion_5.png` | Resultado de `grep -r "token" .` y `grep -r "key" .` |
| `Clonacion_6.png` | Resultado de `grep -r "http" .` |

---

## Procedimiento

### 1. Verificación de HTTrack

```bash
httrack --version
```

### 2. Clonado del sitio

```bash
httrack "https://latinoamericacomparte.com/" \
  -O "$HOME/ProyectoColobiaComparte" \
  "+*.netlify.app/*" \
  "+*.css" "+*.js" "+*.png" "+*.jpg" "+*.jpeg" "+*.gif" \
  -v
```

| Parámetro | Función |
|---|---|
| `"URL"` | URL objetivo a clonar |
| `-O $HOME/...` | Carpeta destino del espejo |
| `+*.netlify.app/*` | Incluye recursos del dominio netlify asociado |
| `+*.css / +*.js` | Incluye hojas de estilo y scripts |
| `+*.png/jpg/jpeg/gif` | Incluye imágenes |
| `-v` | Modo verbose (detalle del proceso) |

**Resultado del clonado:**

| Métrica | Valor |
|---|---|
| Duración | 1 minuto 26 segundos |
| Links escaneados | 76 |
| Archivos escritos | 66 (≈ 38.8 MB) |
| Velocidad promedio | 245 bytes/seg — 1.4 req/conexión |
| Errores 404 | 9 |
| Advertencias | 0 |

**Rutas con error 404** (referenciadas en el HTML pero sin página independiente):
- `/quienes.html`, `/paises.html`, `/impacto.html`, `/noticias.html`, `/apoyar.html`

> Indica navegación basada en *single-page* (SPA) o anclas de sección, no páginas separadas.

### 3. Levantamiento del servidor local

```bash
cd ~/ProyectoColobiaComparte
python3 -m http.server 8080 --bind 127.0.0.1
```

El espejo quedó disponible en `http://127.0.0.1:8080` con todos los recursos respondiendo `200 OK`.

---

## Análisis de exposición técnica (Fase White Hat)

### `grep -r "api" .`

**Hallazgos:**
- `fonts.googleapis.com` — Google Fonts (servicio público)
- `https://api.web3forms.com/submit` — endpoint del formulario de contacto (servicio de terceros)

**Riesgo:** Bajo. Ambos son servicios públicos y documentados.

---

### `grep -r "fetch" .`

**Hallazgos:**
```
./latinoamericacomparte.com/index.html:
  const response = await fetch(contactFormNew.action, {
./latinoamericacomparte.com/index-2.html:
  const response = await fetch(contactFormNew.action, {
```

La URL se obtiene dinámicamente desde el atributo `action` del formulario. **No hay endpoints hardcodeados.**

**Riesgo:** Bajo. Práctica correcta.

---

### `grep -r "token" .`

**Hallazgos:** Solo la librería `anime.min.js` (CDN de Cloudflare) usa el término `token` internamente como parte de su lógica de animación.

**Riesgo:** Nulo. Código de tercero, sin tokens de autenticación propios.

---

### `grep -r "key" .`

**Hallazgos:** Solo `anime.min.js` con propiedades internas (`keyframes`, `keyCode`, etc.).

**Riesgo:** Nulo. No se encontraron claves API ni secretos.

---

### `grep -r "http" .`

**Hallazgos:** Recursos externos identificados:
- `https://cdnjs.cloudflare.com/ajax/libs/animejs/3.2.1/anime.min.js` — librería de animaciones
- `https://fonts.googleapis.com` — fuentes tipográficas
- `https://api.web3forms.com/submit` — formulario de contacto
- Imágenes y logos alojados en el propio dominio

El caché HTTrack (`hts-cache/old.txt`) confirmó la descarga exitosa de 66 archivos con código `200 OK`.

---

## Resumen de hallazgos

| Patrón buscado | Resultado | Riesgo |
|---|---|---|
| `api` | Google Fonts + web3forms | 🟡 Bajo (servicios públicos) |
| `fetch` | Llamada dinámica al formulario | 🟡 Bajo (sin endpoints hardcodeados) |
| `token` | Solo librería anime.js | ✅ Nulo (código de tercero) |
| `key` | Solo librería anime.js | ✅ Nulo (código de tercero) |
| `http` | CDN, fuentes, logos | 🟡 Bajo (recursos externos conocidos) |

---

## Buenas prácticas (Blue Team)

1. **Nunca poner secretos en el frontend:**
   ```javascript
   // ❌ MAL
   const SECRET_KEY = "abc123";
   // ✅ BIEN — en el backend
   process.env.API_KEY
   ```

2. **Usar tokens dinámicos** — generados por sesión con tiempo de expiración, nunca permanentes.

3. **Configurar CORS correctamente** — solo permitir el propio dominio en `Access-Control-Allow-Origin`.

4. **Implementar Rate Limiting** en todos los endpoints propios para prevenir fuerza bruta, scraping y DDoS básicos. Un servidor bien configurado responde `429 Too Many Requests` ante abuso.

5. **Resolver rutas 404** — las 5 rutas referenciadas en el HTML deben implementarse como páginas, redirecciones o anclas correctas.

---

## Conclusión

El sitio `https://latinoamericacomparte.com` no presenta exposición crítica de información sensible en su código frontend. Sigue buenas prácticas básicas: no expone tokens, claves API ni secretos en el código fuente accesible al cliente. El único punto de mejora inmediata son las rutas con error 404 y la adición de validación del lado del servidor en el endpoint de formulario.
