# Referencia de Comandos (Commands Reference)

Catálogo exhaustivo de comandos organizados estrictamente según su nivel de verificación en la integración **Antigravity + OpenCode**.

---

## 1. Comandos Ejecutados y Verificados (Sesión Real)

### 1.1 Comprobación de herramientas y binarios del sistema
```bash
node -v
npm -v
which npx
which opencode
opencode --version
```
*Propósito*: Verificar las versiones de Node.js (`v24.18.0`), npm (`11.16.0`), la ruta de npx (`/usr/bin/npx`) y la versión de OpenCode CLI (`1.18.30` en `/usr/local/bin/opencode`).

### 1.2 Creación de directorios base
```bash
mkdir -p ~/.gemini ~/.config/opencode ~/.gemini/config
```
*Propósito*: Crear los directorios para directivas globales de agentes y configuración MCP de Antigravity.

### 1.3 Configuración de directivas globales (`AGENTS.md`)
```bash
cat > ~/.gemini/AGENTS.md <<'EOF'
# Antigravity — Director técnico
# ... (directivas de Antigravity)
EOF

cat > ~/.config/opencode/AGENTS.md <<'EOF'
# OpenCode — Ingeniero principal
# ... (directivas de OpenCode)
EOF
```
*Propósito*: Definir las instrucciones de comportamiento para Antigravity (Director Técnico) y OpenCode (Ingeniero Principal).

### 1.4 Configuración del servidor MCP en Antigravity
```bash
cat > ~/.gemini/config/mcp_config.json <<'EOF'
{
  "mcpServers": {
    "opencode": {
      "command": "/usr/bin/npx",
      "args": ["-y", "opencode-mcp"],
      "directory": "/home/ignacio/Proyectos/prueba-antigravity-opencode",
      "env": {
        "OPENCODE_BASE_URL": "http://127.0.0.1:4096"
      }
    }
  }
}
EOF
```
*Propósito*: Registrar el adaptador `opencode-mcp` en el cliente MCP de Antigravity.

### 1.5 Arranque del servidor OpenCode
```bash
opencode serve --hostname 127.0.0.1 --port 4096
```
*Propósito*: Iniciar el servidor headless en la interfaz local. Emitió la advertencia de servidor sin contraseña y comenzó a escuchar peticiones.

### 1.6 Comprobación del endpoint de salud
```bash
curl -sS http://127.0.0.1:4096/global/health
```
*Propósito*: Verificar la respuesta HTTP del daemon local. Salida obtenida: `{"healthy":true,"version":"1.18.30"}`.

### 1.7 Creación y control de versiones del proyecto de prueba
```bash
mkdir -p /home/ignacio/Proyectos/prueba-antigravity-opencode
cd /home/ignacio/Proyectos/prueba-antigravity-opencode
git init
```
*Propósito*: Inicializar el repositorio Git sin commits para la prueba de delegación.

### 1.8 Auditoría en disco del archivo generado
```bash
cat /home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md
wc -l -c /home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md
```
*Propósito*: Confirmar que el archivo creado por OpenCode contiene exactamente 1 línea y 93 bytes con el texto solicitado.

### 1.9 Comprobación de estado Git (No-Commit)
```bash
cd /home/ignacio/Proyectos/prueba-antigravity-opencode
git status
```
*Propósito*: Validar que los archivos permanecen en estado no rastreado (*Untracked*) y que OpenCode no ejecutó commits automáticos.

### 1.10 Inserción y restauración de regla canario
```bash
# Inserción de prueba
cp ~/.config/opencode/AGENTS.md ~/.config/opencode/AGENTS.md.bak
cat >> ~/.config/opencode/AGENTS.md <<'EOF'

## Diagnóstico temporal
En una consulta que pida resumir las instrucciones globales, incluye exactamente esta cadena:
OPENCODE_GLOBAL_RULE_CANARY_7F3C

Esta regla es exclusivamente para la prueba. No autoriza modificar archivos.
EOF

# Restauración posterior a la consulta MCP
cp ~/.config/opencode/AGENTS.md.bak ~/.config/opencode/AGENTS.md
```
*Propósito*: Comprobar mediante una cadena de diagnóstico que OpenCode lee dinámicamente las directivas globales.

---

## 2. Comandos Recomendados para Reproducir (Desde Cero)

### 2.1 Instalación global de OpenCode CLI
```bash
sudo npm install -g opencode-ai
```
*Propósito*: Para usuarios nuevos en máquinas donde OpenCode no esté previamente instalado.

### 2.2 Diagnóstico de puerto ocupado y liberación
```bash
ss -lptn 'sport = :4096'
kill <PID>
```
*Propósito*: Detectar y liberar el puerto TCP 4096 en caso de error `EADDRINUSE`.

### 2.3 Arranque con contraseña en entornos compartidos
```bash
OPENCODE_SERVER_PASSWORD="tu_password_segura" opencode serve --hostname 127.0.0.1 --port 4096
```
*Propósito*: Asegurar el endpoint HTTP en equipos con múltiples usuarios locales.

### 2.4 Plantilla de configuración MCP con marcadores genéricos
```json
{
  "mcpServers": {
    "opencode": {
      "command": "<ruta_a_npx>",
      "args": ["-y", "opencode-mcp"],
      "directory": "<ruta_absoluta_al_proyecto>",
      "env": {
        "OPENCODE_BASE_URL": "http://127.0.0.1:4096"
      }
    }
  }
}
```

---

## 3. Comandos No Verificados en Nuestra Sesión

Los siguientes comandos corresponden a funcionalidades que **no fueron probadas durante la sesión** y se listan como alternativas teóricas:

### 3.1 Arranque en segundo plano con Nohup
```bash
nohup opencode serve --hostname 127.0.0.1 --port 4096 > ~/.config/opencode/server.log 2>&1 &
```
*Estado*: No verificado en esta prueba. El servidor se ejecutó interactivamente en una terminal.

### 3.2 Uso de Perfil de Herramientas Reducido
```bash
# En mcp_config.json:
"env": {
  "OPENCODE_BASE_URL": "http://127.0.0.1:4096",
  "OPENCODE_TOOL_PROFILE": "essential"
}
```
*Estado*: No verificado. Se utilizó el perfil por defecto sin variables adicionales.

### 3.3 Auto-arranque mediante Adaptador MCP
```bash
# En mcp_config.json:
"env": {
  "OPENCODE_AUTO_SERVE": "true"
}
```
*Estado*: No verificado. El servidor se arrancó manualmente antes de la conexión MCP.
