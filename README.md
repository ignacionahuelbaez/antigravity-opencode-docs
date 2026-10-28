# Integración Antigravity + OpenCode (Director Técnico - Ingeniero Principal vía MCP)

Guía técnica, diseño de arquitectura, registro de operaciones y procedimiento de reproducción paso a paso para integrar **Google Antigravity** (Director Técnico / Arquitecto) y **OpenCode** (Ingeniero Principal de Implementación) mediante el protocolo **Model Context Protocol (MCP)**.

> **Aviso de alcance e independencia**: Este proyecto es una guía comunitaria independiente de documentación técnica. No constituye una publicación ni documentación oficial de Google ni del equipo de OpenCode.

---

## 1. ¿Qué es este proyecto?

Este repositorio establece un flujo de trabajo de desarrollo de software asistido por IA basado en la **separación operativa de responsabilidades**:

- **Antigravity (Director Técnico / Arquitecto)**: Analiza requisitos, examina el repositorio en modo lectura, diseña la solución técnica, estructura planes de trabajo modulares y audita las modificaciones y pruebas.
- **OpenCode (Ingeniero Principal)**: Recibe tareas de implementación delimitadas mediante el protocolo MCP y ejecuta los cambios directamente sobre el espacio de trabajo autorizado.

---

## 2. Objetivos y Consideraciones de Seguridad

1. **Separación de roles por directivas**: Antigravity y OpenCode poseen técnicamente capacidades de lectura y ejecución en el entorno local. La división de responsabilidades (Antigravity orquestando y OpenCode programando) se establece mediante directivas de comportamiento en archivos [`AGENTS.md`](AGENTS.md), las cuales orientan el comportamiento de los modelos pero no sustituyen los controles de permisos y aislamiento del sistema operativo.
2. **Mitigación de colisiones**: Al asignar la escritura de código exclusivamente a OpenCode durante la ejecución, se busca minimizar condiciones de carrera y sobreescrituras descoordinadas entre agentes.
3. **Trazabilidad operativa**: Cada tarea despachada mediante MCP genera registros de sesión (`jobId`, tokens consumidos y archivos modificados), facilitando la supervisión humana del avance.

---

## 3. Arquitectura del Flujo

```text
+--------------------------------------------------------------+
|                     USUARIO / DESARROLLADOR                  |
+--------------------------------------------------------------+
                              | (Requerimiento o meta)
                              v
+--------------------------------------------------------------+
|            ANTIGRAVITY (Director Técnico / Arquitecto)        |
|  - Analiza, planifica tareas y audita resultados             |
|  - Instrucciones globales: ~/.gemini/AGENTS.md               |
|  - Configuración MCP: ~/.gemini/config/mcp_config.json       |
+--------------------------------------------------------------+
                              | (Llamada MCP: opencode_run / opencode_ask)
                              v
+--------------------------------------------------------------+
|                ADAPTADOR MCP (opencode-mcp)                  |
|  - Transporte: STDIO (JSON-RPC)                              |
|  - Invocación: /usr/bin/npx -y opencode-mcp                  |
|  - Puente entre el protocolo MCP y la API REST de OpenCode   |
+--------------------------------------------------------------+
                              | (HTTP REST: POST /session /messages)
                              v
+--------------------------------------------------------------+
|                 OPENCODE SERVER (Headless Daemon)            |
|  - Comando: opencode serve --hostname 127.0.0.1 --port 4096  |
|  - Endpoint de salud: http://127.0.0.1:4096/global/health   |
|  - Instrucciones globales: ~/.config/opencode/AGENTS.md      |
|  - Instrucciones locales: <workspace>/AGENTS.md              |
+--------------------------------------------------------------+
                              | (Edición de código y pruebas)
                              v
+--------------------------------------------------------------+
|                   WORKSPACE DEL PROYECTO                     |
|  - Directorio autorizado: /ruta/al/proyecto                   |
|  - Archivos de código, tests y control de versiones          |
+--------------------------------------------------------------+
```

---

## 4. Reproducción Rápida

### Requisitos previos
- Linux (x86_64 o arm64).
- Node.js versión `>= 22.0.0` (verificado con `v24.18.0`), npm (`11.16.0`) y npx (`/usr/bin/npx`).
- OpenCode CLI (`opencode-ai`) versión `1.18.30` o superior instalada (ruta usual: `/usr/local/bin/opencode`).
- Antigravity instalado y operativo.
- Proveedor LLM configurado en OpenCode (ej. OpenRouter).

### Paso 1: Configurar directivas globales (`AGENTS.md`)
```bash
mkdir -p ~/.gemini ~/.config/opencode ~/.gemini/config

# Directivas globales para Antigravity
cat > ~/.gemini/AGENTS.md <<'EOF'
# Antigravity — Director técnico

## Rol principal
Actuar como director técnico, arquitecto y coordinador del trabajo.
La implementación del código se delega a OpenCode mediante MCP.

## Responsabilidades
- Comprender los requisitos e investigar alternativas.
- Diseñar la arquitectura y preparar planes de implementación.
- Dividir el trabajo en tareas concretas y verificables.
- Delegar las tareas de programación a OpenCode usando las herramientas MCP.
- Revisar los resultados de OpenCode, evaluar los riesgos e informar al usuario.

## Protocolo de delegación
- Identificar el directorio absoluto del proyecto autorizado.
- Enviar a OpenCode una tarea concreta con contexto, archivos, restricciones y criterios de aceptación.
- Comprobar el estado y el resultado de la ejecución antes de darla por completada.
- No afirmar que se utilizó OpenCode si no existe evidencia de la llamada MCP correspondiente.
- Si MCP falla o no está disponible, informar el problema y no sustituir silenciosamente a OpenCode en tareas de implementación.
- No permitir que ambos agentes implementen simultáneamente sobre los mismos archivos.
EOF

# Directivas globales para OpenCode
cat > ~/.config/opencode/AGENTS.md <<'EOF'
# OpenCode — Ingeniero principal

## Rol
Implementar las tareas delegadas por Antigravity y las solicitadas directamente por el usuario.

## Antes de implementar
- Confirmar el directorio de trabajo.
- Leer las instrucciones AGENTS.md del proyecto.
- Comprender el alcance y los criterios de aceptación.

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

### Paso 2: Configurar MCP en Antigravity
Cree el archivo `~/.gemini/config/mcp_config.json`:

```json
{
  "mcpServers": {
    "opencode": {
      "command": "/usr/bin/npx",
      "args": ["-y", "opencode-mcp"],
      "directory": "/ruta/absoluta/a/tu/proyecto",
      "env": {
        "OPENCODE_BASE_URL": "http://127.0.0.1:4096"
      }
    }
  }
}
```
*(Nota: Reinicie Antigravity tras crear o editar este archivo para registrar el servidor MCP).*

### Paso 3: Iniciar el servidor OpenCode
En una terminal local:
```bash
opencode serve --hostname 127.0.0.1 --port 4096
```
Salida observada:
```text
Warning: OPENCODE_SERVER_PASSWORD is not set; server is unsecured.
opencode server listening on http://127.0.0.1:4096
```

Verifique el estado de salud:
```bash
curl -sS http://127.0.0.1:4096/global/health
```
Respuesta esperada:
```json
{"healthy":true,"version":"1.18.30"}
```

### Paso 4: Preparar el proyecto local
```bash
mkdir -p /ruta/absoluta/a/tu/proyecto
cd /ruta/absoluta/a/tu/proyecto
git init
```

Copie las reglas locales de trabajo en `AGENTS.md` (ver plantilla en [AGENTS.md](AGENTS.md)).

### Paso 5: Prueba de humo (*Smoke Test*)
En la sesión de Antigravity, solicite:
> *"Delega a OpenCode mediante MCP la creación del archivo `mcp-smoke-test.md` dentro de `/ruta/absoluta/a/tu/proyecto` con la siguiente línea exacta: `Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.`"*

Verifique el archivo resultante en disco:
```bash
cat /ruta/absoluta/a/tu/proyecto/mcp-smoke-test.md
git status
```
El archivo debe contener exactamente esa línea y figurar como *Untracked* en Git (sin commits automáticos).

---

## 5. Índice de Documentación Detallada

Para una revisión exhaustiva de cada fase técnica, consulte los documentos en el directorio `docs/`:

- [**docs/architecture.md**](docs/architecture.md): Arquitectura de componentes, roles de los harnesses, modelo de transporte MCP y flujo entre subsistemas.
- [**docs/setup.md**](docs/setup.md): Guía de instalación, variables de entorno, configuración de MCP y orden estricto de despliegue.
- [**docs/verification.md**](docs/verification.md): Pruebas de humo, endpoint `/global/health`, auditoría Git y prueba de directiva canario (`OPENCODE_GLOBAL_RULE_CANARY_7F3C`).
- [**docs/troubleshooting.md**](docs/troubleshooting.md): Diagnóstico de puertos ocupados, avisos de seguridad en loopback, compatibilidad de Node.js y tareas bloqueadas.
- [**docs/operations-log.md**](docs/operations-log.md): Bitácora cronológica real distinguiendo lo **ejecutado y verificado**, lo **recomendado** y lo **no verificado**.
- [**docs/commands-reference.md**](docs/commands-reference.md): Catálogo completo de comandos categorizados por nivel de verificación.
- [**docs/files-and-config.md**](docs/files-and-config.md): Rutas, contenido de configuraciones y propósitos de cada archivo del sistema.
- [**AGENTS.md**](AGENTS.md): Directivas de prompt para la gobernanza operativa de los agentes.

---

## 6. Referencias Oficiales Consultadas

- [Documentación oficial de OpenCode](https://opencode.ai/docs/)
- [Referencia del servidor headless de OpenCode](https://opencode.ai/docs/server/)
- [Especificación del Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [Repositorio del adaptador opencode-mcp](https://github.com/AlaeddineMessadi/opencode-mcp)
