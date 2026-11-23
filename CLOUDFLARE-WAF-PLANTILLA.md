# TK-04.01 · Plantilla Declarativa de Reglas Cloudflare WAF y Protección de Origen

> **Propósito:** Guía operativa lista para aplicar en la consola de Cloudflare de Solutoday una vez se habilite el acceso externo registrado en [`TK-01.01`](https://trello.com/c/uC10IK1Y).  
> **Complemento en código (`web-v2`):** Los controles de aplicación (`TrustProxies`, `RequireHttps`, `SecurityHeaders` y `RateLimiter`) ya están implementados y probados en el PR de `TK-04.01`.

---

## 1. TLS y Protección del Origen

1. **Modo SSL/TLS en Cloudflare:** `Full (Strict)` con certificado de origen Cloudflare o Let's Encrypt válido en Nginx Proxy Manager.
2. **Always Use HTTPS:** Activo (`ON`) para redirigir HTTP a HTTPS en el borde.
3. **Restricción de origen (AWS Security Group / Firewall EC2):**
   * Permitir tráfico entrante a los puertos `80` y `443` **únicamente desde los rangos IP públicos oficiales de Cloudflare** (IPv4 e IPv6).
   * Mantener el puerto administrativo `81` de Nginx Proxy Manager restringido a `127.0.0.1:81:81` (acceso exclusivo por túnel SSH autorizado).
4. **Variables en `.env` de `web-v2` detrás de Cloudflare + Nginx Proxy Manager:**
   ```dotenv
   TRANSPORT_REQUIRE_HTTPS=true
   TRANSPORT_TRUSTED_PROXIES=<IP_O_CIDR_DEL_PROXY_INTERNO>
   TRANSPORT_TRUST_CLOUDFLARE_HEADERS=true
   TRANSPORT_SECURITY_HEADERS_ENABLED=true
   TRANSPORT_HSTS_MAX_AGE=31536000
   ```

---

## 2. Reglas de WAF y Rate Limiting en Borde (Compatibles con Login, Formularios y Webhooks)

| # | Regla en Cloudflare WAF / Rate Limiting | Expresión recomendada | Acción | Motivo / Compatibilidad |
| :--- | :--- | :--- | :--- | :--- |
| 1 | **Protección de inicio de sesión** | `(http.request.method eq "POST" and (http.request.uri.path eq "/api/login" or http.request.uri.path eq "/user-login" or http.request.uri.path eq "/login"))` | Rate limit: `20 req / 1 min` por IP -> `Managed Challenge` o `429` | Frena fuerza bruta en borde sin bloquear usuarios legítimos tras NAT; Laravel aplica además el límite `auth-login` por IP + cuenta. |
| 2 | **Protección de recuperación de clave** | `(http.request.method eq "POST" and (http.request.uri.path eq "/api/forgot-password" or http.request.uri.path eq "/forgot-password" or http.request.uri.path eq "/reset-password"))` | Rate limit: `10 req / 1 min` por IP -> `Block (429)` | Evita abuso de envío de correos SMTP. |
| 3 | **Protección de formularios públicos** | `(http.request.method eq "POST" and (http.request.uri.path eq "/api/contact-us" or http.request.uri.path eq "/api/check-username" or http.request.uri.path eq "/user-subscribe"))` | Rate limit: `20 req / 1 min` por IP -> `Managed Challenge` | Mitiga spam y enumeración de usuarios sin afectar envíos normales. |
| 4 | **Excepción / continuidad de Webhooks** | `(http.request.method eq "POST" and starts_with(http.request.uri.path, "/api/webhooks/"))` | `Skip` Managed Challenge / Bot Fight Mode (mantener validación de firma en Laravel y `throttle:webhooks`) | Garantiza que las notificaciones automáticas de pasarelas (Stripe / Wompi en Ítem 7) no reciban desafíos JavaScript ni bloqueos falsos. |
| 5 | **Bloqueo de rutas sensibles y archivos ocultos** | `(http.request.uri.path contains "/.env" or http.request.uri.path contains "/.git" or starts_with(http.request.uri.path, "/install"))` | `Block (403)` | Defensa en profundidad complementaria al retiro de `/install/*` en el PR #1. |

---

## 3. Verificación pendiente cuando Solutoday entregue acceso (`TK-01.01`)

- [ ] Confirmar plan contratado en Cloudflare (Free / Pro / Business) y dominios activos (`solutoday.com`, `api.solutoday.com`, QA).
- [ ] Aplicar las 5 reglas anteriores y verificar `Full (Strict)` + restricción de origen en AWS EC2.
- [ ] Ejecutar prueba acotada y autorizada verificando que login, formularios y webhooks legítimos siguen respondiendo `200 OK`.
