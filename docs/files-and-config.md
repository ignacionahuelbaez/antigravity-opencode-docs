# Catálogo de Archivos y Configuraciones (Files and Config)

Este documento detalla todas las rutas, archivos de configuración, directivas de gobernanza y artefactos creados y verificados durante la integración de **Antigravity** y **OpenCode**.

---

## 1. Mapa General de Rutas

| Ruta en el Entorno Real | Marcador Genérico | Tipo | Componente | Propósito |
|---|---|---|---|---|
| `/home/ignacio/.gemini/config/mcp_config.json` | `~/.gemini/config/mcp_config.json` | Configuración JSON | Antigravity | Registro del servidor MCP `opencode` y parámetros de transporte. |
| `/home/ignacio/.gemini/AGENTS.md` | `~/.gemini/AGENTS.md` | Directivas Markdown | Antigravity | Instrucciones de comportamiento para Antigravity como Director Técnico. |
| `/home/ignacio/.config/opencode/AGENTS.md` | `~/.config/opencode/AGENTS.md` | Directivas Markdown | OpenCode | Instrucciones de comportamiento para OpenCode como Ingeniero Principal. |
| `/home/ignacio/Proyectos/prueba-antigravity-opencode/AGENTS.md` | `<ruta_proyecto>/AGENTS.md` | Directivas Markdown | Ambos agentes | Contrato local de reglas y flujo de trabajo para el proyecto. |
| `/home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md` | `<ruta_proyecto>/mcp-smoke-test.md` | Artefacto de prueba | Auditoría | Archivo creado por OpenCode vía MCP durante la prueba de humo. |
| `/home/ignacio/.local/state/opencode-mcp/tasks/` | `~/.local/state/opencode-mcp/tasks/` | Base de datos JSON | Adaptador MCP | Almacén persistente de trabajos (*Task Store*) con registros de ejecución. |
| `/usr/local/bin/opencode` | `<ruta_a_opencode>` | Binario ejecutable | Sistema Operativo | CLI oficial del motor OpenCode (`v1.18.30`). |
| `/usr/bin/npx` | `<ruta_a_npx>` | Binario ejecutable | Sistema Operativo | Ejecutor de paquetes Node.js (`v24.18.0` / npm `11.16.0`). |

---

## 2. Detalle de Archivos de Configuración

### 2.1 Archivo de Servidores MCP de Antigravity
- **Ruta real**: `/home/ignacio/.gemini/config/mcp_config.json`
- **¿Por qué existe?**: Antigravity lo consulta al iniciar para inicializar servidores MCP externos.
- **Contenido real verificado**:
  ```json
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
  ```
- **Parámetros utilizados**:
  - `command`: `/usr/bin/npx` (ruta canónica al ejecutable de Node).
  - `args`: `["-y", "opencode-mcp"]` (descarga y ejecución automática del adaptador).
  - `directory`: Directorio de trabajo absoluto del proyecto asignado.
  - `OPENCODE_BASE_URL`: Endpoint HTTP del daemon headless de OpenCode.
- **Herramientas detectadas**: Tras cargar este archivo, Antigravity expuso **87 herramientas** bajo el prefijo `opencode_`. Esta cifra corresponde a la versión `3.0.0` del adaptador y puede variar en futuras versiones.

---

### 2.2 Directivas Globales de Antigravity
- **Ruta real**: `/home/ignacio/.gemini/AGENTS.md`
- **Naturaleza**: Instrucciones de sistema (prompt) que orientan el comportamiento de Antigravity. No representan una barrera de permisos a nivel de sistema operativo.
- **Puntos clave**:
  - Asigna a Antigravity el rol de Director Técnico / Arquitecto.
  - Instruye a no modificar directamente código fuente si OpenCode está disponible.
  - Exige el uso obligatorio de herramientas MCP para tareas de implementación.

---

### 2.3 Directivas Globales de OpenCode
- **Ruta real**: `/home/ignacio/.config/opencode/AGENTS.md`
- **Naturaleza**: Instrucciones de comportamiento inyectadas en cada sesión de OpenCode.
- **Puntos clave**:
  - Asigna a OpenCode el rol de Ingeniero Principal.
  - Exige confirmar el directorio de trabajo y leer el `AGENTS.md` del proyecto.
  - Prohíbe operaciones destructivas, commits automáticos y modificaciones fuera de scope.

---

### 2.4 Directivas Locales del Proyecto de Prueba
- **Ruta real**: `/home/ignacio/Proyectos/prueba-antigravity-opencode/AGENTS.md`
- **Contenido verificado**:
  ```markdown
  # Reglas de trabajo: Antigravity + OpenCode

  ## 1. Roles

  ### Antigravity — Director técnico
  - Comprender los objetivos y requisitos del usuario.
  - Investigar y analizar las alternativas.
  - Diseñar la arquitectura y elaborar planes de implementación.
  - Dividir el trabajo en tareas concretas.
  - Delegar las implementaciones a OpenCode mediante las herramientas MCP.
  - Revisar los resultados y comunicar problemas, riesgos y decisiones pendientes.

  ### OpenCode — Ingeniero principal
  - Implementar las tareas que Antigravity le delegue.
  - Modificar el código y los archivos necesarios para la tarea.
  - Ejecutar las pruebas y verificaciones pertinentes.
  - Informar los archivos modificados, las pruebas realizadas y los resultados.
  - No ampliar el alcance de la tarea sin autorización.

  ## 2. Flujo de trabajo

  1. El usuario define el objetivo.
  2. Antigravity analiza el problema y prepara un plan.
  3. Si la tarea requiere cambios en archivos o código, Antigravity solicita aprobación del plan cuando corresponda.
  4. Antigravity delega la implementación a OpenCode mediante MCP.
  5. OpenCode implementa y verifica los cambios.
  6. Antigravity revisa el resultado y presenta un informe al usuario.

  ## 3. Uso obligatorio de la delegación

  - Antigravity no debe implementar directamente los cambios de código que corresponden a OpenCode.
  - Para implementar, debe utilizar las herramientas reales disponibles del servidor MCP `opencode`.
  - Debe indicar explícitamente el directorio absoluto del proyecto al delegar tareas.
  - Si la herramienta MCP no está disponible o falla, debe informar el problema y no simular una delegación exitosa.
  - Antigravity puede leer archivos y revisar diferencias para analizar o verificar el trabajo.

  ## 4. Protección del proyecto

  - Trabajar únicamente dentro del directorio del proyecto autorizado.
  - No crear otros proyectos ni cambiar de directorio de trabajo por iniciativa propia.
  - No borrar archivos, ejecutar commits ni modificar configuraciones globales sin autorización.
  - No ejecutar comandos destructivos ni operaciones que afecten otros proyectos.
  - No modificar archivos ajenos a la tarea solicitada.
  - No hacer que ambos agentes implementen simultáneamente sobre los mismos archivos.

  ## 5. Verificación y comunicación

  - No afirmar que una tarea se completó sin evidencia.
  - Informar qué herramientas se utilizaron y qué archivos cambiaron.
  - Distinguir las pruebas ejecutadas de las que quedaron pendientes.
  - Si hay errores, falta de permisos o dudas sobre el alcance, detenerse y explicarlo.
  - Priorizar cambios pequeños, verificables y fáciles de revertir.
  ```

---

### 2.5 Artefacto de Prueba de Humo
- **Ruta real**: `/home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md`
- **Contenido verificado**:
  ```text
  Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.
  ```
- **Tamaño verificado**: 1 línea, 93 bytes.

---

### 2.6 Registros de Auditoría del Task Store (Jobs de OpenCode)
- **Ruta real**: `/home/ignacio/.local/state/opencode-mcp/tasks/a047c5e2f68e44fc996ceca6f59d225133ea5c0a48fb3feeef1091d29ed7c2a6/`
- **Archivos verificados**:
  - `job_5cfb8a36761a4de2996d5759a4ebc3d7.json`: Creación del archivo `mcp-smoke-test.md` (3934 tokens input, 39 output).
  - `job_a5d349b0df094c389a732dbf452a8073.json`: Comprobación de unicidad de contenido.
  - `job_4ca6338bb47548fe815159f4ceb3e5d8.json`: Lectura e interpretación de `AGENTS.md` por parte de OpenCode.
