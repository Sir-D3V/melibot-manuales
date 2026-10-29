# melibot — manuales de operación

Manuales para el servicio que responde automáticamente las preguntas de Mercado Libre Venezuela (n8n + Gmail + OpenRouter + melibot con Chrome en una PC de oficina).

- [Manual 1 — Levantar el servicio cuando se apaga o reinicia la PC](MANUAL-1-reiniciar-servicio.md)
- [Manual 2 — Instalar el servicio en otra PC (primera vez)](MANUAL-2-instalacion-nueva-pc.md)

## Valores a sustituir

| Placeholder | Significado |
|---|---|
| `<IP_VPS>` | IP o dominio del servidor donde corren n8n y melibot |
| `<USUARIO_VPS>` | Usuario SSH que administra `/opt/apps/melibot` |
| `<USUARIO_ADMIN>` | Usuario del VPS con permisos `sudo` |
| `<CORREO_GMAIL>` | Cuenta de Gmail que recibe las notificaciones de Mercado Libre |
| `<DOMINIO_N8N>` | Dominio donde está publicado n8n |

Los scripts `office-pc/` que mencionan los manuales y los workflows de n8n no están en este repositorio.
