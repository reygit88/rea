# REA en Claude Code (Windows) — guía de instalación

Registro de la instalación hecha el 2026-10-25 con el método TriCore (evidencia primero, cambios pequeños).
Esta guía vive en una rama propia del fork `reygit88/rea`; no modifica el código original.

## 1. Qué es REA

REA (*Reverse Engineer Anything*) es un servidor MCP local que permite a un agente inspeccionar software, desde el comportamiento de una app hasta binarios nativos. El análisis se ejecuta en el propio equipo.

- Proyecto original: <https://github.com/morluto/rea> (licencia MIT, se conserva en `LICENSE`)
- Paquete npm: `rea-agents`
- Copia en este fork: `origin` = `reygit88/rea`, `upstream` = `morluto/rea`

Úsalo solo sobre software que tengas derecho a analizar.

## 2. Versiones comprobadas

| Componente | Versión |
|---|---|
| Windows | 11 Pro 10.0.26300 (PowerShell 5.1) |
| Node.js | v24.19.0 |
| npm | 11.17.0 |
| Git | 2.55.0.windows.3 |
| Claude Code | 2.1.270 |
| REA instalado (npm) | `rea-agents@6.7.0` |
| Tag del código fuente de esa versión | `rea-agents-6.7.0` = `27dc42d10249f0410904a75c0997842eb03b931c` |
| Código fuente clonado (`main`) | `bb17e90f27dc8908c1c91c05055679678768c266` |

**La versión instalada y el código clonado difieren.** `main` está 1712 commits por delante del tag 6.7.0 y aún no está publicado en npm. Lo que se ejecuta es el paquete publicado 6.7.0, no el clon.

### Requisito de Node

- La documentación de `main` (no publicada) pide Node 22.x ≥ 22.23.4, 24.x ≥ 24.22.1 o 26.12.1+.
- El paquete publicado 6.7.0 y su documentación (tag `rea-agents-6.7.0`) aceptan 22.x ≥ 22.19, 24.x ≥ 24.11 y 26+. Node 24.19.0 cumple, así que no se tocó Node.
- Recomendación pendiente (no aplicada): actualizar Node 24 a ≥ 24.22.1 más adelante, sobre todo si se actualiza REA a una versión nueva.

## 3. Comandos comprobados (en orden)

Todos desde una carpeta vacía ajena al repositorio. `npm_config_omit=optional` evita descargar Frida (ver §9).

```powershell
$env:npm_config_omit = 'optional'

# 1) Revisar el paquete sin ejecutarlo (verifica integridad contra el registro)
npm pack rea-agents@6.7.0 --ignore-scripts

# 2) Simulación: solo muestra el plan, no escribe nada
npx -y rea-agents@6.7.0 setup --client claude_code --dry-run --json

# 3) Aplicar (solo Claude Code), tras copiar ~/.claude.json
npx -y rea-agents@6.7.0 setup --client claude_code --yes --json

# 4) Diagnóstico oficial
npx -y rea-agents@6.7.0 doctor --client claude_code --skill --json

# 5) Claude Code reconoce el servidor
claude mcp get rea
claude mcp list
```

Resultado de la verificación de integridad: el SHA-512 del tarball descargado coincide con el del registro npm. El paquete no define scripts `preinstall`/`install`/`postinstall` propios.

## 4. Qué cambió en el equipo

| Elemento | Detalle |
|---|---|
| `%USERPROFILE%\.claude.json` | Se añadió solo `mcpServers.rea` (alcance usuario): `cmd.exe /d /c npx -y rea-agents@6.7.0 mcp`. El resto de claves y proyectos quedó igual. |
| `%USERPROFILE%\.claude\skills\reverse-engineer-anything` | Skill oficial de REA (7 archivos). |
| `%USERPROFILE%\.claude.json.rea.backup` | Copia automática de REA (primer estado). |
| `%USERPROFILE%\.claude.json.tricore-backup-20261025` | Copia propia hecha antes de aplicar. |
| `%LOCALAPPDATA%\npm-cache\_npx\<hash>` | Caché de npx con `rea-agents@6.7.0`. |

No se configuraron Claude Desktop, Codex, Cursor, Copilot CLI ni VS Code.

## 5. Uso desde Claude Code

1. **Reinicia Claude Code** para que cargue la integración (un MCP nuevo no aparece en una sesión ya abierta).
2. Comprueba con `claude mcp list` que `rea` figura como `Connected`.
3. Pide en lenguaje natural, por ejemplo: «Usa REA para analizar la aplicación Electron en `C:\ruta\a\la\app` y lista sus canales IPC y endpoints».
4. La skill `reverse-engineer-anything` guía los flujos. El servidor expone 207 herramientas; la principal para JavaScript/Electron es `analyze_javascript_application` (parámetro obligatorio `input_path`).
5. También hay CLI: `npx -y rea-agents@6.7.0 analyze-javascript-application C:\ruta\absoluta --json`.

## 6. Prueba funcional realizada

Objetivo: demostrar que REA analiza un objetivo real. App Electron mínima y sintética (sin datos personales) con tres canales IPC (`app:get-version`, `notes:save`, `shell:open-docs`), un puente `contextBridge` (`notesApi`) y un endpoint `https://api.example.com/v1`.

- **CLI** (`analyze-javascript-application ... --json`): salida 0, evidencia de ~439 KB, `confidence: derived`, `authority: shipped-artifact`.
- **MCP real** (handshake stdio con el mismo comando que usa Claude Code, `tools/call`): `serverInfo = rea 6.7.0`, 207 herramientas, `isError: false`, mismos hallazgos.
- Hallazgos presentes en la evidencia: los 3 canales IPC, `api.example.com`, `notesApi`, `contextBridge`, `contextIsolation`, `nodeIntegration`, `openExternal`.
- No apareció como cadena literal la URL `https://example.com/docs` del `openExternal`.

Niveles de estado:

| Nivel | Estado |
|---|---|
| Instalado | Sí |
| Conectado | Sí (`claude mcp list`: `Connected`; `doctor`: registro y skill alineados) |
| Prueba funcional superada | Sí, **solo** para el análisis estático de JavaScript/Electron |
| Invocación desde una sesión interactiva de Claude Code | **No realizada** (requiere reiniciar Claude Code) |
| Motores nativos (Ghidra, IDA, JADX, Kuna, Hopper) | No instalados ni probados |

## 7. Actualizar

REA queda fijado a una versión en la configuración (`rea-agents@6.7.0`), así que no se actualiza solo.

```powershell
# Según la documentación oficial (no ejecutado en esta instalación):
npx -y rea-agents@latest setup --client claude_code
```

`setup` muestra el plan y pide aprobación; haz antes una copia de `%USERPROFILE%\.claude.json` y reinicia Claude Code después. Antes de subir de versión, lee las notas de la versión y el requisito de Node (§2).

Para sincronizar este fork con el original sin sobrescribir nada:

```powershell
git fetch upstream --tags
git switch main
git merge --ff-only upstream/main
git push origin main
```

## 8. Desinstalar

Oficial (según la documentación, no ejecutado en esta instalación):

```powershell
npx -y rea-agents@6.7.0 uninstall
# y, si quieres borrar también caché y estado de REA:
npx -y rea-agents@6.7.0 uninstall --purge-data
```

En 6.7.0, `uninstall` solo acepta `--purge-data` (no admite `--client`) y retira las configuraciones y skills propias de REA en todos los clientes.

Manual y acotado a Claude Code:

```powershell
claude mcp remove rea -s user
Remove-Item -Recurse "$env:USERPROFILE\.claude\skills\reverse-engineer-anything"
```

Si algo sale mal, restaura `%USERPROFILE%\.claude.json` desde `.claude.json.tricore-backup-20261025`. La caché de npx se puede borrar sin riesgo.

## 9. Limitaciones y funciones opcionales

- Sin motor nativo, REA hace análisis **estático** de JavaScript/Electron, .NET, firmas APK, PyInstaller, Flutter y metadatos de Go/Rust/IL2CPP.
- Análisis de binarios nativos: requiere **Ghidra** (en Windows x64 es experimental: Ghidra 12.1.x, JDK 21+, `JAVA_HOME` y `GHIDRA_INSTALL_DIR`), **IDA** (licencia comercial propia) o **Hopper** (no disponible en Windows). REA no los instala.
- **JADX** (Android) y **Kuna** requieren un JAR/ejecutable aportado por el usuario (`REA_JADX_MCP_JAR`, `REA_KUNA_COMMAND`).
- **Frida** (instrumentación dinámica) es una dependencia opcional de npm que se omitió a propósito; esas funciones no están disponibles.
- `doctor` informa `environment_healthy: false` porque no hay motores nativos configurados; es lo esperado en esta instalación mínima.
- Coste: el núcleo es gratuito (MIT). IDA y otras herramientas comerciales requieren licencia propia. No se ha contratado nada.
- El primer arranque del MCP descarga dependencias a la caché de npx; si Claude Code muestra un fallo de conexión en frío, repite `claude mcp list` (la caché ya estará caliente).
