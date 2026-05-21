# DNS Footprinting Pasivo — latinoamericacomparte.com

**Herramientas:** `nslookup` · `dig` · `whois`  
**Entorno:** Kali Linux (terminal bash)  
**Estudiante:** Juan Diego Velasco Quintero  
**Fecha:** Tunja, 20 de mayo de 2026  

> Análisis pasivo — sin contacto directo con el servidor objetivo. Cumplimiento de la Ley 1273 de 2009 (Colombia).

---

## Descripción

El DNS Footprinting Pasivo es la fase inicial de reconocimiento en la que se recopila información pública sobre la infraestructura DNS de un objetivo consultando bases de datos públicas (servidores DNS, registros WHOIS). No se generó ningún paquete directo hacia el servidor del sitio.

---

## Pasos ejecutados

### Paso 1 y 2 — `nslookup`: Resolución, NS y Reverse DNS

Se ejecutaron cuatro consultas:

```bash
# Registro A — IP del servidor
nslookup latinoamericacomparte.com

# Registro NS — servidores de nombres
nslookup -type=ns latinoamericacomparte.com

# DNS inverso (PTR)
nslookup 64.207.187.143
```

**Resultados:**

| Consulta | Resultado | Significado |
|---|---|---|
| Registro A | `64.202.187.143` | IP pública del servidor |
| Registro NS | `ns05/ns06.domaincontrol.com` | GoDaddy administra el DNS |
| PTR (primera) | `copilots.dev` | Otro dominio comparte la IP (shared hosting) |
| PTR (segunda) | Sin nombre (solo ARPA) | PTR no configurado |

---

### Paso 3 — `dig`: Consulta A y Trazado Jerárquico (`+trace`)

```bash
# Consulta directa con TTL y tiempo de respuesta
dig latinoamericacomparte.com

# Trazado completo desde los servidores raíz
dig -4 +trace latinoamericacomparte.com
```

**Resultados:**

| Campo | Valor | Significado |
|---|---|---|
| Status | `NOERROR` | Consulta exitosa |
| TTL | 159 segundos | Caché expira en ~2.6 min (atípicamente bajo) |
| IP | `64.202.187.143` | Servidor del objetivo |
| Query time | 7 ms | Respuesta rápida |
| Servidores raíz | `a–m.root-servers.net` | 13 raíces del DNS global |
| TLD `.com` | `gtld-servers.net` | Servidores GTLD de Verisign |

---

### Paso 4 — `dig`: DNSKEY y DS (Verificación DNSSEC)

```bash
dig DNSKEY latinoamericacomparte.com
dig DS latinoamericacomparte.com
```

Ambas consultas devolvieron `ANSWER: 0` — sin registros de seguridad.

| Consulta | ANSWER | Conclusión |
|---|---|---|
| `DNSKEY` | 0 registros | Sin claves DNSSEC publicadas |
| `DS` | 0 registros | Sin delegación segura desde `.com` |
| SOA serial | `2025110701` | Última actualización de zona: 1 nov 2025 |

---

### Paso 5 — `whois`: Registro del Dominio

```bash
whois latinoamericacomparte.com
```

| Campo WHOIS | Valor |
|---|---|
| Registrar | GoDaddy.com, LLC |
| Creation Date | 2025-08-06 |
| Registry Expiry Date | 2026-08-06 |
| Name Servers | NS05/NS06.DOMAINCONTROL.COM |
| DNSSEC | `unsigned` |
| Domain Status | 4× `clientProhibited` (protección activa) |

---

## Hallazgos

| Hallazgo | Nivel | Detalle |
|---|---|---|
| DNSSEC no implementado | 🔴 Alto | Vulnerable a DNS Cache Poisoning |
| TTL demasiado bajo (159 s) | 🟠 Medio | Riesgo de DNS rebinding |
| Dominio joven (< 1 año) | 🟠 Medio | Baja reputación en filtros de correo |
| Shared hosting en /24 | 🟠 Medio | `copilots.dev` en la misma IP |
| Expira agosto 2026 | 🟠 Medio | Renovación próxima — riesgo de domain squatting |
| Contacto admin expuesto | 🟡 Info | `dns@jomax.net` visible en SOA |
| Protección EPP activa | ✅ Buena práctica | 4× `clientProhibited` — evita domain hijacking |

---

## Recomendaciones

1. **Implementar DNSSEC** — firmar criptográficamente las respuestas DNS para prevenir cache poisoning.
2. **Normalizar el TTL** — subir de 159 s a un valor estándar (3600–86400 s) para mayor estabilidad.
3. **Monitorear la expiración** — el dominio vence en agosto 2026; renovar con anticipación para evitar domain squatting.
