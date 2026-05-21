# Maltego — Reconocimiento OSINT e Infraestructura

**Herramienta:** Maltego CE (Community Edition)  
**Entorno:** Kali Linux 2026.1 — Oracle VirtualBox amd64  
**Estudiante:** Juan Diego Velasco Quintero  
**Fecha:** Tunja, 20 de mayo de 2026  

> Reconocimiento 100% pasivo mediante consultas a fuentes públicas (DNS, WHOIS, Wayback Machine). Sin tráfico directo al servidor. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción

Maltego es una plataforma OSINT que visualiza relaciones entre entidades (dominios, IPs, servidores de nombres, correos, redes sociales, certificados) a través de un grafo interactivo. Cada relación se descubre ejecutando *transforms*, que son consultas automáticas a fuentes públicas.

**Transforms usados:** DNS · MX · NS · WWW · Wayback Machine · Netblock

---

## Capturas y vistas analizadas

### Vista 1 — Grafo completo de infraestructura

Se ejecutaron transforms sobre el dominio raíz `latinoamericacomparte.com`, generando un grafo con todos los subdominios DNS, servidores de nombres, IPs y presencia en redes sociales.

**Entidades descubiertas:**

| Entidad | Tipo | Relevancia |
|---|---|---|
| `www.latinoamericacomparte.com` | WWW | Sitio principal |
| `aulavirtual.latinoamericacomparte.com` | DNS | Plataforma Moodle |
| `cpanel.latinoamericacomparte.com` | DNS | Panel de control web |
| `whm.latinoamericacomparte.com` | DNS | WHM (gestor de hosting) |
| `webdisk.latinoamericacomparte.com` | DNS | Acceso WebDAV |
| `www.admin.latinoamericacomparte.com` | DNS | Panel admin expuesto |
| `ns05/ns06.domaincontrol.com` | NS | GoDaddy |
| `64.202.187.143` | IP | Dirección del servidor |
| `www.instagram.com` | Red social | Presencia en Instagram |

---

### Vista 2 — Subdominios y Wayback Machine

El grafo expandido reveló registros históricos de Wayback Machine y el servidor de correo MX.

| Entidad | Tipo | Detalle |
|---|---|---|
| `mail.latinoamericacomparte.com` | MX | Servidor de correo |
| `webmail.latinoamericacomparte.com` | DNS | Webmail público |
| `dns@jomax.net` | Email | Contacto GoDaddy/registrador |
| 2026 Mar 07: Wayback URL | Histórico | Última captura mar 2026 |
| 2026 Feb 15: `http://...` | Wayback URL | Capturas HTTP sin TLS (×2) |

---

### Vista 3 — Bloque de red e IP

Grafo enfocado en la relación entre la IP y su bloque de red, más el Autonomous System.

| Campo | Valor |
|---|---|
| IP del servidor | `64.202.187.143` |
| Rango del bloque | `64.202.187.0/24` (Blocksize 256) |
| Sistema Autónomo | AS398101 |
| Servicios en la IP | cPanel, WHM, Moodle (misma dirección) |

---

## Hallazgos

| Hallazgo | Nivel | Detalle |
|---|---|---|
| cPanel/WHM expuesto en DNS | 🔴 Alto | Subdominios públicos revelan tecnología de gestión |
| Panel admin visible en DNS | 🔴 Alto | `www.admin.` accesible públicamente |
| Shared hosting `/24` | 🟠 Medio | 256 hosts en mismo AS398101 |
| Capturas HTTP sin TLS | 🟠 Medio | Tráfico plano en feb 2026 |
| Wayback Machine URLs | 🟠 Medio | Versiones históricas accesibles |
| Registrador GoDaddy expuesto | 🟠 Medio | `dns@jomax.net` en grafo |
| Instagram vinculado | 🟡 Info | Presencia en redes sociales confirmada |

---

## Recomendaciones

1. **Ocultar subdominios administrativos** — `cpanel.`, `whm.`, `www.admin.` no deberían ser resolubles públicamente vía DNS; configurar vistas DNS internas o usar acceso VPN.
2. **Infraestructura centralizada** — mover servicios críticos (correo, panel, Moodle) a IPs separadas reduce el radio de impacto de un ataque exitoso.
3. **Gestión de huella digital (OPSEC)** — revisar regularmente qué información del dominio es visible públicamente y reducir la superficie de reconocimiento pasivo.
