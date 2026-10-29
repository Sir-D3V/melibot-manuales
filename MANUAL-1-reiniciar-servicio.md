# Manual 1 — Levantar melibot cuando se apaga o reinicia la PC

Aplica a la PC de la oficina (Linux). Al apagarla se caen **dos** piezas locales: el Chrome del bot y el túnel SSH. El VPS (n8n, melibot, base de datos) sigue funcionando 24/7 y no hay que tocarlo.

## Resumen rápido

```bash
bash ~/meli-bot/office-pc/1-abrir-chrome-melibot.sh
nohup bash ~/meli-bot/office-pc/2-tunel-vps.sh > /tmp/tunel-melibot.log 2>&1 &
ssh <USUARIO_VPS>@<IP_VPS> 'docker exec melibot node -e "fetch(\"http://127.0.0.1:3000/health\").then(r=>r.text()).then(console.log)"'
```
El último comando debe mostrar `"cdp":"connected"` y `"alerts":true`. Los pasos detallados siguen abajo.

## Qué pasa mientras la PC está apagada
- Si llega un correo de "Te preguntaron en…", n8n lo detecta, pero melibot no encuentra el Chrome.
- Telegram te avisa: "Chrome de la oficina no disponible".
- n8n reintenta 3 veces, esperando 7 minutos entre intentos. Si sigue sin Chrome, la ejecución queda en rojo (**❌ Requiere intervención humana**).
- No se pierde nada: la pregunta sigue pendiente en Mercado Libre y se puede reintentar (paso 5).

## Paso 1 — Encender la PC e iniciar sesión en el escritorio
El Chrome del bot necesita pantalla gráfica. No funciona sin sesión de escritorio iniciada.

## Paso 2 — Abrir el Chrome del bot
En una terminal:
```bash
bash ~/meli-bot/office-pc/1-abrir-chrome-melibot.sh
```
- Se abre Chrome en mercadolibre.com.ve con el perfil dedicado `~/melibot-chrome-profile`.
- La sesión de Mercado Libre queda guardada en ese perfil. Normalmente **no hay que iniciar sesión otra vez**.
- Los mensajes de error de GPU/OpenGL en la terminal son normales e inofensivos.
- **Deja esa ventana de Chrome abierta**. Puedes minimizarla, pero no cerrarla. No hace falta maximizarla.
- No uses ese Chrome para navegar a otros sitios: es el Chrome del bot.

Si Chrome pide iniciar sesión (la sesión caducó), inicia sesión en Mercado Libre con la cuenta del vendedor, incluyendo 2FA o captcha si aparecen.

Comprobación: debe responder JSON.
```bash
curl -s http://127.0.0.1:9222/json/version | head -3
```

## Paso 3 — Abrir el túnel SSH hacia el VPS
Opción A, en segundo plano (recomendada):
```bash
nohup bash ~/meli-bot/office-pc/2-tunel-vps.sh > /tmp/tunel-melibot.log 2>&1 &
```
Opción B, en primer plano: ejecuta `bash ~/meli-bot/office-pc/2-tunel-vps.sh` en una terminal y déjala abierta.

El script borra el socket viejo del VPS, abre el túnel y se reconecta solo cada 10 s si la conexión cae.

Comprobación:
```bash
tail -5 /tmp/tunel-melibot.log
pgrep -af "ssh -N"
```
Debe haber un proceso `ssh -N ... -R /opt/apps/melibot/cdp/chrome.sock:127.0.0.1:9222`. Si aparece `Permission denied`, la llave SSH de esta PC no está autorizada en el VPS (ver el Manual 2, paso 4).

## Paso 4 — Verificar que todo está conectado
```bash
ssh <USUARIO_VPS>@<IP_VPS> 'docker exec melibot node -e "fetch(\"http://127.0.0.1:3000/health\").then(r=>r.text()).then(console.log)"'
```
Resultado esperado:
```json
{"ok":true,"dryRun":false,"mode":"connect","cdp":"connected","blocked":null,"alerts":true}
```

| Campo | Valor correcto | Si no |
|---|---|---|
| `cdp` | `connected` | `unreachable`: Chrome o túnel no están arriba (pasos 2 y 3) |
| `alerts` | `true` | `false`: falta `TELEGRAM_CHAT_ID` en el `.env` del VPS |
| `blocked` | `null` | Hay un captcha o sesión cerrada en cooldown: resuélvelo en el Chrome y espera 7 min |
| `dryRun` | `false` | `true`: el bot lee y redacta, pero **no publica** |

## Paso 5 — Reintentar los correos que llegaron con la PC apagada
Solo si llegaron preguntas mientras estaba apagada:
1. Abre `https://<DOMINIO_N8N>` → workflow **melibot — Respuestas automáticas Mercado Libre** → pestaña **Executions**.
2. Las ejecuciones en rojo son preguntas sin responder. Abre cada una y pulsa **Retry**.
3. Si el Retry falla (n8n no reintenta ejecuciones que ya terminaron bien), responde esa pregunta a mano en la Central de vendedores, o haz que llegue un correo nuevo.

Los workflows de n8n no hay que activarlos de nuevo: siguen publicados.

## Qué hacer ante alertas de Telegram

| Alerta | Qué hacer |
|---|---|
| Captcha detectado | Ve al Chrome del bot, resuelve el captcha en Mercado Libre y espera. El bot no toca Mercado Libre durante 7 min y luego reintenta solo. |
| Sesión cerrada | Inicia sesión en Mercado Libre en el Chrome del bot. |
| Chrome no disponible | Repite los pasos 2 y 3. |

## Problemas frecuentes

| Síntoma | Causa | Solución |
|---|---|---|
| `cdp: unreachable` con Chrome abierto | Túnel caído | Paso 3 |
| `cdp: unreachable` con túnel arriba | Chrome sin `--remote-debugging-port` (abierto a mano) | Ciérralo y ábrelo con el script del paso 2 |
| El script 1 dice "No se encontro Chrome" | Chrome no instalado | Instala `google-chrome-stable` |
| El túnel repite "Tunel caido" | Sin internet o llave SSH no autorizada | Revisa la conexión; `ssh <USUARIO_VPS>@<IP_VPS> true` debe responder sin pedir contraseña |
| `ssh: connect ... refused` | VPS inaccesible | Revisa la red; el VPS responde en `<IP_VPS>:22` |

## Reiniciar el servicio en el VPS (casi nunca hace falta)
Se hace como usuario `<USUARIO_VPS>`, no como `<USUARIO_ADMIN>`:
```bash
ssh <USUARIO_VPS>@<IP_VPS>
cd /opt/apps/melibot
docker compose up -d --force-recreate
docker compose logs --tail 20 melibot
```

## Detener el bot a propósito
- Para que no publique: en el VPS, pon `DRY_RUN=true` en `/opt/apps/melibot/.env` y ejecuta `docker compose up -d --force-recreate`.
- Para que no procese correos: en n8n, despublica el workflow principal.
