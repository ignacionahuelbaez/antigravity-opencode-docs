# Procedimientos de Verificación y Auditoría

Este documento reúne las comprobaciones de conectividad, pruebas de humo (*smoke tests*), auditorías de integridad y evidencias empíricas recolectadas durante la validación del sistema integrado **Antigravity + OpenCode**.

---

## 1. Verificación del Endpoint de Salud (`/global/health`)

Antes de realizar llamadas MCP, se comprobó la disponibilidad del servicio headless directamente desde la terminal del sistema operativo.

### 1.1 Comando ejecutado en consola:
```bash
curl -sS http://127.0.0.1:4096/global/health
```

### 1.2 Salida observada:
```json
{"healthy":true,"version":"1.18.30"}
```

**Resultado de la comprobación**: El servidor se encuentra activo, respondiendo en la interfaz loopback y confirmando la versión `1.18.30`.

---

## 2. Detección del Servidor MCP en Antigravity

Tras reiniciar Antigravity con el archivo `~/.gemini/config/mcp_config.json` configurado, Antigravity detectó y expuso dinámicamente las herramientas del servidor.

### 2.1 Evidencia observada:
- **Herramientas detectadas**: Se registraron **87 herramientas** bajo el prefijo `opencode_` (correspondientes a los esquemas JSON de `opencode-mcp@3.0.0`).
- **Nota técnica**: Esta cifra de 87 herramientas refleja el catálogo expuesto por esa versión concreta y no debe considerarse un requisito numérico estático para futuras actualizaciones.
- **Herramienta de diagnóstico**: La herramienta MCP `opencode_health` realiza internamente una petición al endpoint `/global/health` del servidor OpenCode.

---

## 3. Prueba de Humo de Delegación (*Smoke Test*)

Se configuró un espacio de trabajo de prueba con las siguientes propiedades:
- **Ruta del proyecto**: `/home/ignacio/Proyectos/prueba-antigravity-opencode`
- **Control de versiones**: `git init` (sin commits previos).
- **Directivas locales**: Archivo `AGENTS.md` en la raíz del proyecto.

### 3.1 Delegación solicitada a Antigravity:
Antigravity recibió la instrucción de delegar la creación del archivo `mcp-smoke-test.md` con contenido prefijado, sin realizar commits y sin alterar otros archivos.

### 3.2 Llamada MCP realizada por Antigravity:
Antigravity ejecutó la herramienta `opencode_run` con los siguientes parámetros:

```json
{
  "ServerName": "opencode",
  "ToolName": "opencode_run",
  "Arguments": {
    "directory": "/home/ignacio/Proyectos/prueba-antigravity-opencode",
    "prompt": "Crear el archivo mcp-smoke-test.md con el contenido exacto: 'Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.' No modificar otros archivos ni realizar commits."
  }
}
```

### 3.3 Metadatos del Trabajo Registrado en el Task Store:
- **Ruta de almacenamiento**:
  `~/.local/state/opencode-mcp/tasks/a047c5e2f68e44fc996ceca6f59d225133ea5c0a48fb3feeef1091d29ed7c2a6/job_5cfb8a36761a4de2996d5759a4ebc3d7.json`
- **Job ID**: `job_5cfb8a36761a4de2996d5759a4ebc3d7`
- **Session ID**: `ses_e9e4e77bcffe0aHFbdW579Io96` (Slug: `neon-island`)
- **Proveedor y Modelo**: `openrouter` / `nvidia/nemotron-3.5-super-vl:free`
- **Respuesta reportada por OpenCode**:
  > *"Done. I created only the mcp-smoke-test.md file with the exact content requested. No commits, no other files modified.*
  > *Archivo creado: `mcp-smoke-test.md` con el contenido exacto solicitado. No se creó ni modificó ningún otro archivo, y no se ejecutaron commits.*
  > *_cost: $0.0000 | tokens: 3934 in, 39 out, 27 reasoning_"*

---

## 4. Auditoría de Archivos y Control de Versiones

A diferencia de la respuesta reportada por MCP, las siguientes validaciones se ejecutaron **directamente en consola**:

### 4.1 Comprobación del contenido generado:
```bash
cat /home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md
```
**Salida observada**:
```text
Integración MCP verificada: OpenCode recibió y ejecutó una tarea delegada por Antigravity.
```

### 4.2 Comprobación de líneas y bytes:
```bash
wc -l -c /home/ignacio/Proyectos/prueba-antigravity-opencode/mcp-smoke-test.md
```
**Resultado**: 1 línea, 93 bytes.

### 4.3 Auditoría de estado Git (Comprobación de No-Commit):
```bash
cd /home/ignacio/Proyectos/prueba-antigravity-opencode
git status
```
**Salida observada en consola**:
```text
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	AGENTS.md
	mcp-smoke-test.md

nothing added to commit but untracked files present (use "git add" to track)
```
**Conclusión de auditoría**: OpenCode escribió en el sistema de archivos del workspace asignado y no ejecutó commits automáticos, cumpliendo la directiva de seguridad establecida en `AGENTS.md`.

---

## 5. Prueba de Directiva Canario (`OPENCODE_GLOBAL_RULE_CANARY_7F3C`)

Para validar empíricamente si OpenCode lee y procesa las instrucciones globales de `~/.config/opencode/AGENTS.md`, se realizó una prueba controlada con una cadena canario:

### 5.1 Inserción de la regla temporal:
Se respaldó `~/.config/opencode/AGENTS.md` y se añadió una directiva de diagnóstico:

```bash
cp ~/.config/opencode/AGENTS.md ~/.config/opencode/AGENTS.md.bak

cat >> ~/.config/opencode/AGENTS.md <<'EOF'

## Diagnóstico temporal
En una consulta que pida resumir las instrucciones globales, incluye exactamente esta cadena:
OPENCODE_GLOBAL_RULE_CANARY_7F3C

Esta regla es exclusivamente para la prueba. No autoriza modificar archivos.
EOF
```

### 5.2 Consulta vía MCP:
Se consultó a OpenCode a través de MCP para que resumiera sus directivas de trabajo.

### 5.3 Resultado observado:
OpenCode incluyó en su respuesta la cadena exacta `OPENCODE_GLOBAL_RULE_CANARY_7F3C`, demostrando que el contenido del archivo de instrucciones es inyectado e interpretado durante la inferencia.

### 5.4 Restauración de las directivas originales:
Una vez verificada la prueba, se restauró el archivo original para eliminar la regla temporal:

```bash
cp ~/.config/opencode/AGENTS.md.bak ~/.config/opencode/AGENTS.md
# o reescritura del archivo limpio
```

---

## 6. Cuadro de Resumen de Verificaciones

| Comprobación | Canal de Verificación | Resultado Obtenido | Estado |
|---|---|---|:---:|
| **Salud del Servidor** | Consola local (`curl`) | `{"healthy":true,"version":"1.18.30"}` | **Verificado** |
| **Herramientas MCP** | Antigravity Runtime | 87 herramientas detectadas bajo `opencode_` | **Verificado** |
| **Creación de Archivo** | Reporte MCP + Consola (`cat`) | `mcp-smoke-test.md` con texto exacto | **Verificado** |
| **No-Commit no autorizado** | Consola local (`git status`) | Archivos en *Untracked*, 0 commits | **Verificado** |
| **Inyección de Directivas** | Reporte MCP (Canario) | Cadena `OPENCODE_GLOBAL_RULE_CANARY_7F3C` devuelta | **Verificado** |
| **Restauración de Reglas** | Consola local (`cat`) | Archivo limpio sin regla canario residual | **Verificado** |
| **Arranque automático daemon** | — | No configurado ni probado en la sesión | **No verificado** |
