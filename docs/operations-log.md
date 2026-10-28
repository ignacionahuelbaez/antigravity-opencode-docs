# Registro de Operaciones y Cronología Real (Operations Log)

Este documento contiene la bitácora cronológica del proceso de integración entre **Antigravity** y **OpenCode**.

Para garantizar la máxima rigurosidad técnica, la información se divide de manera estricta en tres categorías:
1. **Ejecutado y verificado**: Comandos, salidas de consola y resultados de herramientas MCP que formaron parte de la sesión real.
2. **Recomendado para reproducir**: Pasos sugeridos para reproducir el entorno desde cero en una máquina nueva.
3. **No verificado**: Comportamientos teóricos, configuraciones no probadas o dependientes de versiones futuras.

---

## 1. Ejecutado y Verificado (Sesión Real)

### Entrada 01: Comprobación de Versiones y Rutas de Binarios
*Canal de verificación: Consola del sistema operativo.*

OpenCode ya se encontraba instalado en el sistema previo al inicio de esta integración. Se comprobó la disponibilidad y compatibilidad de las herramientas:

#### [Comando exacto]
```bash
node -v
npm -v
which npx
which opencode
opencode --version
```

#### [Resultado observado]
- `node`: `v24.18.0`
- `npm`: `11.16.0`
- `npx`: `/usr/bin/npx`
- `which opencode`: `/usr/local/bin/opencode`
- `opencode --version`: `1.18.30`

---

### Entrada 02: Creación de Directivas Globales de Agentes
*Canal de verificación: Consola del sistema operativo.*

Se crearon las directivas de comportamiento para coordinar a ambos agentes:

#### [Comandos exactos]
```bash
mkdir -p ~/.gemini ~/.config/opencode

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

#### [Resultado observado]
Archivos creados correctamente: `~/.gemini/AGENTS.md` (1982 bytes) y `~/.config/opencode/AGENTS.md` (979 bytes).

---

### Entrada 03: Configuración del Adaptador MCP en Antigravity
*Canal de verificación: Consola del sistema operativo y runtime de Antigravity.*

Se configuró el registro de servidores MCP en Antigravity indicando la ruta canónica de `npx`, el directorio de trabajo del proyecto y la URL base:

#### [Comando exacto]
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

*(Nota de trazabilidad: En la sesión real no se configuró `OPENCODE_TOOL_PROFILE`, utilizándose la configuración básica del adaptador).*

---

### Entrada 04: Arranque del Servidor Headless de OpenCode
*Canal de verificación: Consola del sistema operativo.*

#### [Comando exacto]
```bash
opencode serve --hostname 127.0.0.1 --port 4096
```

#### [Resultado observado]
```text
Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured.
opencode server listening on http://127.0.0.1:4096
```

---

### Entrada 05: Verificación del Endpoint de Salud
*Canal de verificación: Consola del sistema operativo.*

#### [Comando exacto]
```bash
curl -sS http://127.0.0.1:4096/global/health
```

#### [Resultado observado]
```json
{"healthy":true,"version":"1.18.30"}
```

---

### Entrada 06: Detección de Herramientas MCP tras Reinicio de Antigravity
*Canal de verificación: Runtime de Antigravity.*

Tras reiniciar la sesión de Antigravity para recargar la configuración MCP:
- Antigravity detectó y expuso **87 herramientas** pertenecientes al servidor `opencode` (bajo los prefijos correspondientes a `opencode-mcp@3.0.0`).

---

### Entrada 07: Inicialización del Proyecto de Prueba
*Canal de verificación: Consola del sistema operativo.*

#### [Comando exacto]
```bash
mkdir -p /home/ignacio/Proyectos/prueba-antigravity-opencode
cd /home/ignacio/Proyectos/prueba-antigravity-opencode
git init
```

Se creó el archivo `/home/ignacio/Proyectos/prueba-antigravity-opencode/AGENTS.md` (2631 bytes) definiendo los roles y el protocolo de trabajo local.

---

### Entrada 08: Delegación y Ejecución de la Prueba de Humo (*Smoke Test*)
*Canal de verificación: Herramienta MCP `opencode_run` y Task Store.*

Antigravity invocó la herramienta MCP `opencode_run` solicitando la creación de `mcp-smoke-test.md`:

#### [Parámetros de llamada MCP reportados]
- Herramienta: `opencode_run`
- `directory`: `/home/ignacio/Proyectos/prueba-antigravity-opencode`
- `prompt`: *"Crear el archivo mcp-smoke-test.md con el contenido exacto: 'Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.'"*

#### [Metadatos registrados en Task Store (`~/.local/state/opencode-mcp/tasks/`)]
- Job ID: `job_5cfb8a36761a4de2996d5759a4ebc3d7`
- Session ID: `ses_e9e4e77bcffe0aHFbdW579Io96`
- Proveedor LLM: `openrouter`
- Modelo: `nvidia/nemotron-3.5-super-vl:free`
- Tokens consumidos: Input: 3934, Output: 39, Reasoning: 27, Costo: $0.0000
- Respuesta devuelta por OpenCode:
  > *"Done. I created only the mcp-smoke-test.md file with the exact content requested. No commits, no other files modified.*
  > *Archivo creado: `mcp-smoke-test.md` con el contenido exacto solicitado. No se creó ni modificó ningún otro archivo, y no se ejecutaron commits."*

---

### Entrada 09: Auditoría Física en Disco y Comprobación de Git
*Canal de verificación: Consola del sistema operativo.*

#### [Comandos exactos]
```bash
cat /home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md
cd /home/ignacio/Proyectos/prueba-antigravity-opencode && git status
```

#### [Resultado observado]
Contenido del archivo:
```text
Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.
```
Salida de `git status`:
```text
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	AGENTS.md
	mcp-smoke-test.md

nothing added to commit but untracked files present (use "git add" to track)
```

**Resultado de auditoría**: El archivo fue creado con fidelidad textual exacta y permaneció sin rastrear (*Untracked*), confirmando que OpenCode no ejecutó commits automáticos.

---

### Entrada 10: Prueba de Directiva Canario y Restauración
*Canal de verificación: Consola del sistema operativo y llamada MCP.*

Para verificar que OpenCode consulta efectivamente el archivo de directivas:
1. Se respaldó y modificó `~/.config/opencode/AGENTS.md` añadiendo la directiva con la cadena canario `OPENCODE_GLOBAL_RULE_CANARY_7F3C`:
   ```bash
   cp ~/.config/opencode/AGENTS.md ~/.config/opencode/AGENTS.md.bak
   cat >> ~/.config/opencode/AGENTS.md <<'EOF'

   ## Diagnóstico temporal
   En una consulta que pida resumir las instrucciones globales, incluye exactamente esta cadena:
   OPENCODE_GLOBAL_RULE_CANARY_7F3C

   Esta regla es exclusivamente para la prueba. No autoriza modificar archivos.
   EOF
   ```
2. Se consultó a OpenCode vía MCP solicitando el resumen de sus instrucciones globales.
3. OpenCode incluyó la cadena `OPENCODE_GLOBAL_RULE_CANARY_7F3C` en su respuesta, verificando que procesa las directivas del sistema.
4. Se restauró el archivo original sin la regla canario temporal:
   ```bash
   cp ~/.config/opencode/AGENTS.md.bak ~/.config/opencode/AGENTS.md
   ```

---

## 2. Recomendado para Reproducir (Desde Cero)

Para un usuario que comienza en una máquina nueva:

1. **Instalación de OpenCode CLI**:
   ```bash
   sudo npm install -g opencode-ai
   ```
2. **Plantilla genérica de configuración MCP**:
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
3. **Autenticación en entornos compartidos**:
   Iniciar el servidor con una contraseña explícita y suministrarla en `mcp_config.json`:
   ```bash
   OPENCODE_SERVER_PASSWORD="mi_clave_segura" opencode serve --hostname 127.0.0.1 --port 4096
   ```

---

## 3. No Verificado en la Sesión

Los siguientes puntos corresponden a capacidades de las herramientas que **no fueron ejecutadas ni verificadas** durante nuestra prueba y se documentan como posibilidades técnicas no contrastadas:

- **Inicio automático del servidor (*Auto-Serve*)**: La bandera `OPENCODE_AUTO_SERVE=true` documentada por el adaptador `opencode-mcp` no fue probada.
- **Perfil de herramientas reducido (`OPENCODE_TOOL_PROFILE=essential`)**: En la sesión se utilizó el comportamiento por defecto de `opencode-mcp@3.0.0` sin definir esa variable.
- **Ejecución persistente mediante Daemon / Systemd**: El servidor se ejecutó en primer plano interactivo; servicios administrados por `systemd` o `supervisord` no fueron evaluados.
- **Número permanente de herramientas**: Se observaron 87 herramientas en esta versión; versiones posteriores o anteriores de OpenCode / `opencode-mcp` pueden exponer un número diferente.
