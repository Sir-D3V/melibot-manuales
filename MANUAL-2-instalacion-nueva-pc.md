# Manual 2 — Instalar melibot en otra PC (primera vez)

## Cómo está armado el sistema
```
Gmail ──► n8n (VPS) ──► melibot (Docker, VPS) ──túnel SSH──► Chrome (PC de la oficina) ──► Mercado Libre
              │                                                        (IP residencial)
              └─► OpenRouter (redacta)        Alertas ──► Telegram
```
- n8n, melibot y la base de datos viven en el **VPS** y ya están instalados.
- Una **PC nueva** solo necesita Chrome con depuración remota, el túnel SSH y una sesión iniciada en Mercado Libre.
- Mercado Libre debe verse siempre desde la IP de esa PC (residencial), nunca desde el VPS.

Reglas de oro:
- **Solo una PC a la vez** puede tener el túnel abierto. Antes de activar la nueva, detén Chrome y el túnel en la anterior.
- No hay que reinstalar nada en el VPS ni cambiar workflows de n8n al cambiar de PC.

## Requisitos
- Linux con escritorio, o Windows 10/11 (trae `ssh.exe`).
- Google Chrome instalado.
- Internet estable; la PC debe estar encendida y con la sesión de escritorio abierta mientras el bot trabaje.
- Acceso al VPS con un usuario que pueda ejecutar `sudo` (por ejemplo `<USUARIO_ADMIN>`), solo para autorizar la llave en el paso 4.
- Los scripts de `office-pc/` del repositorio.

---

## Paso 1 — Instalar Chrome
**Linux (Debian/Ubuntu/Elementary):**
```bash
wget -O /tmp/chrome.deb https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo apt install -y /tmp/chrome.deb
```
**Windows:** instala Chrome desde google.com/chrome.

## Paso 2 — Copiar los scripts a la PC nueva
Opciones:
- Desde el VPS: `scp -r <USUARIO_VPS>@<IP_VPS>:/opt/apps/melibot/office-pc ~/melibot-office` (necesita la llave del paso 4 ya autorizada, así que haz primero ese paso o copia por USB).
- Por USB o red desde la PC actual: la carpeta `~/meli-bot/office-pc/`.

Contenido:
- Linux: `1-abrir-chrome-melibot.sh` y `2-tunel-vps.sh`
- Windows: `1-abrir-chrome-melibot.bat` y `2-tunel-vps.bat`

En Linux, adapta las rutas de los comandos de este manual si no usas `~/meli-bot/office-pc/`.

## Paso 3 — Generar la llave SSH de esta PC
**Linux:**
```bash
ssh-keygen -t ed25519 -f ~/.ssh/melibot_vps -N ""
```
**Windows (PowerShell):**
```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\melibot_vps -N '""'
```
- La passphrase debe quedar **vacía**: el túnel se reconecta solo y no puede pedirla.
- Se crean `melibot_vps` (privada, **no la compartas**) y `melibot_vps.pub` (pública, no es secreta).

Los scripts conectan con `ssh <USUARIO_VPS>@<IP_VPS>` sin indicar llave. Para que usen la nueva, agrega esto a `~/.ssh/config` (en Windows, `%USERPROFILE%\.ssh\config`):
```
Host <IP_VPS>
  HostName <IP_VPS>
  User <USUARIO_VPS>
  IdentityFile ~/.ssh/melibot_vps
  IdentitiesOnly yes
```

## Paso 4 — Autorizar la llave en el VPS
Muestra la llave pública:
```bash
cat ~/.ssh/melibot_vps.pub        # Windows: type %USERPROFILE%\.ssh\melibot_vps.pub
```
Entra al VPS con tu usuario administrador y agrégala al usuario `<USUARIO_VPS>`:
```bash
ssh <USUARIO_ADMIN>@<IP_VPS>
echo 'PEGA_AQUI_LA_LLAVE_PUBLICA' | sudo tee -a /home/<USUARIO_VPS>/.ssh/authorized_keys
```
Prueba desde la PC nueva (no debe pedir contraseña):
```bash
ssh <USUARIO_VPS>@<IP_VPS> echo OK
```
La primera vez acepta la huella del servidor escribiendo `yes`.

Para revocar una PC más adelante, borra su línea de `/home/<USUARIO_VPS>/.ssh/authorized_keys`.

## Paso 5 — Detener la PC anterior
En la PC anterior:
```bash
pkill -f 2-tunel-vps.sh; pkill -f "ssh -N"
```
Y cierra su Chrome del bot. En Windows, cierra la ventana del `.bat` del túnel.

## Paso 6 — Abrir el Chrome del bot e iniciar sesión
**Linux:**
```bash
bash ~/melibot-office/1-abrir-chrome-melibot.sh
```
**Windows:** doble clic en `1-abrir-chrome-melibot.bat`.

En la ventana que se abre (perfil nuevo y vacío):
1. Inicia sesión en **Mercado Libre** con la cuenta **vendedora** (la que recibe las preguntas). Puede pedir 2FA o captcha: resuélvelo tú.
2. Confirma que ves el panel de vendedor, por ejemplo abriendo `https://vendedores.mercadolibre.com.ve`.
3. Deja la ventana abierta y minimizada. No navegues a otros sitios con ese Chrome.

La sesión se guarda en `~/melibot-chrome-profile` (Windows: `%USERPROFILE%\melibot-chrome-profile`). Es solo para el bot: no la sincronices ni la uses para otras cosas. Mercado Libre puede pedir una verificación extra por entrar desde un equipo nuevo.

Comprobación (debe devolver JSON):
```bash
curl -s http://127.0.0.1:9222/json/version | head -3
```
En Windows PowerShell: `curl http://127.0.0.1:9222/json/version`.

## Paso 7 — Abrir el túnel
**Linux (en segundo plano):**
```bash
nohup bash ~/melibot-office/2-tunel-vps.sh > /tmp/tunel-melibot.log 2>&1 &
tail -5 /tmp/tunel-melibot.log
```
**Windows:** doble clic en `2-tunel-vps.bat`. Deja la ventana abierta.

Debe mostrar "Abriendo tunel..." y quedarse corriendo sin errores. Si ves `Permission denied (publickey)`, repite los pasos 3 y 4.

## Paso 8 — Verificar el servicio
Desde la PC nueva:
```bash
ssh <USUARIO_VPS>@<IP_VPS> 'docker exec melibot node -e "fetch(\"http://127.0.0.1:3000/health\").then(r=>r.text()).then(console.log)"'
```
Esperado:
```json
{"ok":true,"dryRun":false,"mode":"connect","cdp":"connected","blocked":null,"alerts":true}
```
- `cdp: connected` confirma que el VPS ve el Chrome de esta PC.
- `alerts: true` confirma que Telegram está configurado.

## Paso 9 — Prueba controlada
1. Para probar sin publicar, en el VPS pon `DRY_RUN=true` en `/opt/apps/melibot/.env` y ejecuta `docker compose up -d --force-recreate` como `<USUARIO_VPS>`.
2. Haz una pregunta desde la cuenta de pruebas en una publicación del vendedor.
3. En n8n → **Executions** debe aparecer la ejecución en verde con la respuesta redactada y `status: dry_run`.
4. Pon `DRY_RUN=false` y recrea el contenedor para que publique de verdad.
5. Haz otra pregunta de prueba. Debe quedar respondida en Mercado Libre.

Con `DRY_RUN=false`, cualquier pregunta real se responde sola y es irreversible: Mercado Libre solo permite una respuesta.

## Arranque diario
Cada vez que se apague la PC, sigue el **Manual 1**.

---

## Apéndice A — Instalación completa en un VPS nuevo
Solo hace falta si se pierde el VPS o se migra. Tiene prerrequisitos que no se cubren aquí en detalle.

**Requisitos del VPS:** Docker y Docker Compose, la red externa `proxy-net`, n8n corriendo en esa red, y un usuario `<USUARIO_VPS>` (uid 1001) en el grupo `docker`.

1. Copia el proyecto a `/opt/apps/melibot` (sin `node_modules` ni `.env`) y deja `data/`, `profile/` y `cdp/` con dueño `<USUARIO_VPS>`:
   ```bash
   rsync -av --exclude node_modules --exclude .env --exclude data --exclude profile ~/meli-bot/ <USUARIO_VPS>@<IP_VPS>:/opt/apps/melibot/
   ssh <USUARIO_VPS>@<IP_VPS> 'cd /opt/apps/melibot && mkdir -p data profile cdp'
   ```
2. Crea el `.env`:
   ```bash
   cd /opt/apps/melibot && cp .env.example .env && chmod 600 .env
   ```
   Rellena estas variables (un valor por línea, sin comentarios al final):
   - `MELIBOT_API_TOKEN`: genera uno con `openssl rand -hex 32`; el mismo valor va en la credencial de n8n.
   - `TELEGRAM_BOT_TOKEN`: lo da @BotFather.
   - `TELEGRAM_CHAT_ID`: el ID numérico de tu chat; escríbele antes al bot y consúltalo con @userinfobot.
   - `N8N_ALERT_WEBHOOK_URL`: la URL interna del webhook del workflow de alertas.
   - Mantén `DRY_RUN=true`, `BROWSER_MODE=connect` y `MELIBOT_UID=1001`/`MELIBOT_GID=1001`.
3. Levanta como `<USUARIO_VPS>` (no como `<USUARIO_ADMIN>`: el `.env` es modo 600 y solo lo lee `<USUARIO_VPS>`):
   ```bash
   docker compose up -d --build
   docker compose logs --tail 20 melibot
   ```
4. En n8n, importa los dos workflows de `n8n/`: el principal y el de alertas.
5. Crea las credenciales y asígnalas a sus nodos:
   - **Gmail OAuth2** en el nodo del Gmail Trigger.
   - **Header Auth "melibot"** con `Authorization: Bearer <MELIBOT_API_TOKEN>`, en los nodos "melibot: leer pregunta" y "melibot: publicar".
   - **Header Auth "OpenRouter"** con `Authorization: Bearer sk-or-...`, en el nodo "OpenRouter: redactar". Debe ser una API key normal, no una key de administración.
   - **Telegram** en los dos nodos del workflow de alertas.
   La guía detallada está en `n8n/README.md`.
6. Guarda cada workflow y pulsa **Publish**. El de alertas primero.
7. Continúa desde el paso 3 de este manual (llave SSH, Chrome, túnel).

## Apéndice B — Datos de referencia

| Elemento | Valor |
|---|---|
| VPS | `<IP_VPS>`, usuario `<USUARIO_VPS>` |
| Directorio del servicio | `/opt/apps/melibot` |
| Socket del túnel | `/opt/apps/melibot/cdp/chrome.sock` |
| Puerto de depuración de Chrome | `127.0.0.1:9222` |
| Perfil de Chrome del bot | `~/melibot-chrome-profile` |
| n8n | `https://<DOMINIO_N8N>` |
| Correo que se vigila | `<CORREO_GMAIL>` |
| Espera tras captcha/sesión | 7 minutos, 3 intentos |
| Secretos | `/opt/apps/melibot/.env` (modo 600, dueño `<USUARIO_VPS>`) |
