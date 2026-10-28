# Guía de Instalación y Configuración (Setup)

Esta guía documenta los requisitos previos, los comandos exactos de configuración, las directivas de agentes y el procedimiento ordenado para levantar la integración entre **Antigravity** y **OpenCode** vía **MCP**.

---

## 1. Requisitos Previos y Entorno Verificado

### Entorno verificado en la sesión real:
- **Sistema Operativo**: Linux x86_64.
- **Node.js**: `v24.18.0` (la versión 3.0.0 de `opencode-mcp` exige `>= 22.0.0`).
- **npm**: `11.16.0`.
- **npx**: `/usr/bin/npx`.
- **OpenCode CLI**: `1.18.30` en `/usr/local/bin/opencode` (*previamente instalado antes de comenzar la integración*).

### Recomendado para usuarios desde cero:
Si parte de un sistema nuevo donde OpenCode no esté presente, instálelo con:
```bash
sudo npm install -g opencode-ai
```
Y verifique:
```bash
opencode --version
which opencode
node -v
```

---

## 2. Configuración de Directivas de Agentes (`AGENTS.md`)

Cree las directivas globales para Antigravity y OpenCode. Recuerde que estas directivas son instrucciones de prompt para coordinar a los modelos y no constituyen mecanismos de aislamiento del sistema operativo.

### 2.1 Reglas Globales de Antigravity (`~/.gemini/AGENTS.md`)
```bash
mkdir -p ~/.gemini

cat > ~/.gemini/AGENTS.md <<'EOF'
# Antigravity — Director técnico

## Rol principal
Actuar como director técnico, arquitecto y coordinador del trabajo.
La implementación del código se delega a OpenCode mediante MCP.

## Responsabilidades
- Comprender los requisitos e investigar alternativas.
- Diseñar la arquitectura y preparar planes de implementación.
- Dividir el trabajo en tareas concretas y verificables.
- Delegar las tareas de programación a OpenCode usando las herramientas
  MCP realmente disponibles en el servidor `opencode`.
- Revisar los resultados de OpenCode, evaluar los riesgos e informar
  al usuario sobre los cambios y las pruebas.

## Protocolo de delegación
- Identificar el directorio absoluto del proyecto autorizado.
- Enviar a OpenCode una tarea concreta con el contexto necesario,
  los archivos relevantes, las restricciones y los criterios de aceptación.
- Utilizar el parámetro de directorio cuando la herramienta lo admita.
- Comprobar el estado y el resultado de la ejecución antes de darla
  por completada.
- No afirmar que se utilizó OpenCode si no existe evidencia de la
  llamada MCP correspondiente.
- Si MCP falla o no está disponible, informar el problema y no
  sustituir silenciosamente a OpenCode en tareas de implementación.
- No permitir que ambos agentes implementen simultáneamente sobre
  los mismos archivos.

## Trabajo propio permitido
- Investigar, analizar y planificar.
- Leer archivos y revisar diferencias.
- Crear o actualizar planes, especificaciones y documentación cuando
  corresponda a la tarea.
- Revisar la implementación realizada por OpenCode.

## Seguridad y control
- Respetar el alcance autorizado y las reglas particulares del proyecto.
- Solicitar aprobación antes de operaciones destructivas, commits,
  cambios globales o acciones externas con consecuencias importantes.
- No modificar proyectos distintos del autorizado.
- Informar con claridad los errores, las limitaciones y las pruebas
  que no se hayan realizado.
EOF
```

### 2.2 Reglas Globales de OpenCode (`~/.config/opencode/AGENTS.md`)
```bash
mkdir -p ~/.config/opencode

cat > ~/.config/opencode/AGENTS.md <<'EOF'
# OpenCode — Ingeniero principal

## Rol
Implementar las tareas delegadas por Antigravity y las solicitadas
directamente por el usuario.

## Antes de implementar
- Confirmar el directorio de trabajo.
- Leer las instrucciones AGENTS.md del proyecto.
- Comprender el alcance y los criterios de aceptación.
- Consultar al usuario si falta información importante.

## Implementación
- Modificar únicamente los archivos necesarios para la tarea.
- Respetar la arquitectura y las convenciones del proyecto.
- No trabajar en otros proyectos por iniciativa propia.
- No realizar commits ni operaciones destructivas sin autorización.
- No modificar configuraciones globales sin autorización.

## Verificación
- Ejecutar las pruebas pertinentes cuando sea posible.
- Informar qué archivos se modificaron y por qué.
- Indicar las pruebas ejecutadas y sus resultados.
- No afirmar que una prueba pasó si no se ejecutó.
- Comunicar errores, limitaciones y decisiones pendientes.
EOF
```

---

## 3. Configuración del Servidor MCP en Antigravity

Antigravity registra los servidores MCP a través de su archivo de configuración global `~/.gemini/config/mcp_config.json`.

### 3.1 Configuración ejecutada en la sesión real:
```bash
mkdir -p ~/.gemini/config

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

### 3.2 Plantilla genérica para reproducir en otro entorno:
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

> **Nota sobre variables opcionales del adaptador `opencode-mcp`**:
> El adaptador soporta variables adicionales como `OPENCODE_TOOL_PROFILE` (`full` o `essential`) o `OPENCODE_AUTO_SERVE`. **En nuestra integración real no se configuró `OPENCODE_TOOL_PROFILE`**, operando con los valores por defecto del paquete.

---

## 4. Arranque del Servidor OpenCode

El servidor headless de OpenCode debe iniciarse de forma explícita antes de interactuar desde Antigravity.

### 4.1 Comando verificado en consola:
```bash
opencode serve --hostname 127.0.0.1 --port 4096
```

### Salida observada:
```text
Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured.
opencode server listening on http://127.0.0.1:4096
```

### Consideración de seguridad:
El aviso `Warning: OPENCODE_SERVER_PASSWORD is not set` indica que la API HTTP no requiere credenciales de autenticación. Escuchar en `127.0.0.1` aísla las peticiones frente a la red externa, pero cualquier proceso local puede interactuar con el endpoint. En entornos compartidos se recomienda configurar una contraseña en el servidor y transferirla mediante `OPENCODE_SERVER_PASSWORD` en el archivo MCP.

### 4.2 Verificación de salud HTTP:
```bash
curl -sS http://127.0.0.1:4096/global/health
```
Salida observada:
```json
{"healthy":true,"version":"1.18.30"}
```

*(Nota sobre arranque en segundo plano: En la sesión real el servidor se ejecutó directamente en una terminal interactiva. Opciones como `nohup` o servicios `systemd` son alternativas posibles no verificadas en esta prueba).*

---

## 5. Orden Riguroso de Pasos

1. **Verificar binarios y dependencias**: Confirmar `Node.js >= 22`, `npx` y `opencode`.
2. **Definir directivas globales**: Crear `~/.gemini/AGENTS.md` y `~/.config/opencode/AGENTS.md`.
3. **Escribir configuración MCP**: Crear `~/.gemini/config/mcp_config.json`.
4. **Levantar el daemon OpenCode**: Ejecutar `opencode serve --hostname 127.0.0.1 --port 4096`.
5. **Verificar endpoint de salud**: Consultar `curl -sS http://127.0.0.1:4096/global/health`.
6. **Reiniciar / Recargar Antigravity**: Para que reconozca el nuevo servidor MCP y registre sus herramientas (en nuestra sesión se detectaron 87 herramientas).
7. **Inicializar espacio de trabajo de prueba**: Crear carpeta, inicializar Git y añadir el `AGENTS.md` de proyecto.
8. **Ejecutar prueba de humo**: Delegar la creación del archivo de prueba desde Antigravity vía MCP.
