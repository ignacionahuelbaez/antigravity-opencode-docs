# Guía de Resolución de Problemas (Troubleshooting)

Este documento recopila las incidencias más comunes, sus causas raíz y los procedimientos técnicos para diagnosticarlas y resolverlas.

---

## 1. Puerto 4096 Ocupado o Conflicto de Red (`EADDRINUSE`)

### 1.1 Síntoma
Al iniciar el servidor con `opencode serve`, la consola muestra:
```text
Error: listen EADDRINUSE: address already in use 127.0.0.1:4096
```

### 1.2 Causa
Existe otra instancia de `opencode` o una aplicación previa utilizando el puerto TCP `4096`.

### 1.3 Solución
1. Identifique el Process ID (PID) que retiene el puerto:
   ```bash
   ss -lptn 'sport = :4096'
   # o bien:
   lsof -i :4096
   ```
2. Finalice el proceso:
   ```bash
   kill <PID>
   ```
   *(Si el proceso no finaliza tras unos segundos, use: `kill -9 <PID>`)*.
3. Si requiere utilizar otro puerto (ej. `4097`):
   - Inicie OpenCode indicando el puerto:
     ```bash
     opencode serve --hostname 127.0.0.1 --port 4097
     ```
   - Actualice la variable `OPENCODE_BASE_URL` en `~/.gemini/config/mcp_config.json`:
     ```json
     "OPENCODE_BASE_URL": "http://127.0.0.1:4097"
     ```

---

## 2. Servidor OpenCode No Detectado (`ECONNREFUSED`)

### 2.1 Síntoma
Antigravity reporta errores al ejecutar herramientas MCP, o la salida muestra:
```text
FetchError: request to http://127.0.0.1:4096/global/health failed, reason: connect ECONNREFUSED 127.0.0.1:4096
```

### 2.2 Diagnóstico
El adaptador `opencode-mcp` intenta conectar con el daemon local de OpenCode pero el proceso no está corriendo o no escucha en la interfaz `127.0.0.1`.

### 2.3 Solución
1. Compruebe si el servicio responde:
   ```bash
   curl -sS http://127.0.0.1:4096/global/health
   ```
2. Si no hay respuesta, levante el daemon en una terminal:
   ```bash
   opencode serve --hostname 127.0.0.1 --port 4096
   ```
3. Verifique que no haya colisiones de resolución de red asegurando que el host sea la dirección IPv4 explícita `127.0.0.1`.

---

## 3. Advertencia de Seguridad: Servidor Sin Contraseña

### 3.1 Síntoma
Al iniciar el servidor se observa el aviso:
```text
Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured.
opencode server listening on http://127.0.0.1:4096
```

### 3.2 Contexto técnico
- Esta advertencia es el comportamiento por defecto de OpenCode cuando no se define la variable `OPENCODE_SERVER_PASSWORD`.
- Al estar vinculado a `127.0.0.1`, el servicio no es accesible desde la red externa. No obstante, cualquier proceso que se ejecute en la máquina local tiene acceso sin autenticación a la API de OpenCode.
- **Recomendación para entornos multi-usuario**: Definir una contraseña segura mediante la variable `OPENCODE_SERVER_PASSWORD` al iniciar el servidor y configurarla correspondientemente en la sección `env` de `~/.gemini/config/mcp_config.json`:
  ```json
  "env": {
    "OPENCODE_BASE_URL": "http://127.0.0.1:4096",
    "OPENCODE_SERVER_PASSWORD": "<tu_contraseña>"
  }
  ```

---

## 4. Incompatibilidad de Versión de Node.js (< 22.0.0)

### 4.1 Síntoma
Al invocar `opencode-mcp` mediante npx, el proceso falla con errores de motor:
```text
npm error code EBADENGINE
npm error package opencode-mcp@3.0.0 requires node >= 22.0.0
```

### 4.2 Causa
La versión 3.0.0 de `opencode-mcp` requiere obligatoriamente **Node.js 22 o superior** (en la sesión real se verificó con `v24.18.0`).

### 4.3 Solución
1. Compruebe la versión actual del sistema:
   ```bash
   node --version
   ```
2. Actualice Node.js mediante su gestor de versiones preferido (ej. NVM):
   ```bash
   nvm install 22
   nvm use 22
   nvm alias default 22
   ```

---

## 5. Tarea MCP en Estado `input_required` o Bloqueada

### 5.1 Síntoma
Una llamada a `opencode_run` permanece en ejecución o el estado del trabajo devuelve:
```json
{
  "status": "input_required"
}
```

### 5.2 Causa
OpenCode solicitó una confirmación o aclaración del usuario (por ejemplo, para ejecutar un comando potencialmente destructivo o no previsto en las directivas).

### 5.3 Solución
1. Inspeccione los registros locales de trabajos:
   ```bash
   ls -la ~/.local/state/opencode-mcp/tasks/*/
   ```
2. Lea el archivo JSON correspondiente al trabajo para revisar qué solicita el agente.
3. Responda a la solicitud mediante la herramienta MCP `opencode_job_input`, o cancele la tarea con `opencode_job_cancel` si se desvía del objetivo.

---

## 6. Las Directivas de `AGENTS.md` No Surten Efecto

### 6.1 Síntoma
OpenCode realiza acciones imprevistas (crea archivos fuera de lugar o intenta ejecutar commits).

### 6.2 Causa
- El archivo `AGENTS.md` no se encuentra en la raíz del proyecto autorizado.
- La llamada MCP no especificó la ruta absoluta canónica en el parámetro `directory`.

### 6.3 Solución
1. Confirme la existencia de los archivos de directivas:
   ```bash
   ls -l ~/.gemini/AGENTS.md ~/.config/opencode/AGENTS.md <ruta_proyecto>/AGENTS.md
   ```
2. En las llamadas MCP, pase siempre la ruta absoluta explícita:
   ```json
   { "directory": "/home/ignacio/Proyectos/prueba-antigravity-opencode" }
   ```
   *(Evite rutas con tilde `~/...` o relativas `.`)*.
