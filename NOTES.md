# NOTES.md — INFOCOOP RustDesk Fork

> Memoria persistente del proyecto entre sesiones de Claude.
> Cuando inicies una sesión nueva, pegar este archivo le da a Claude todo el contexto necesario para retomar el trabajo.

---

## 1. Contexto del proyecto

- **Empresa**: INFOCOOP (Paraguay).
- **Producto final**: "Soporte INFOCOOP" — fork rebrandeado de RustDesk para soporte remoto a clientes.
- **Uso**: soporte remoto a clientes que corren Visual FoxPro + PostgreSQL. Se utiliza el **TCP tunneling** de RustDesk para conectar PgAdmin/ODBC a Postgres remoto.
- **Infra propia**:
  - Servidor `hbbs` + `hbbr` propio en Ubuntu.
  - Host: `remote.infocoop.com.py`
  - Public key: `Ly4NYMM5RGFz5jDgJT7IRyidB7L1nnrsXgB4f31Bbe8=`
  - `conntrack` del relay: timeout 5 días en ESTABLISHED (descartado como causa del bug).
- **ISP del operador (cliente RustDesk)**: usa **CGNAT** con timeout de NAT idle ~30s — esta es la causa raíz del bug de tunneling.
- **Plataforma del operador**: Windows (compila con Claude Desktop / GitHub Actions).
- **Plataforma de los clientes asistidos**: Windows.

---

## 2. Objetivos del fork

### Entregable 1 (alta prioridad) — Rebranding
- Cliente Windows con nombre y logo de INFOCOOP.
- Servidores `hbbs`/`hbbr` y public key preconfigurados (no requiere configuración manual al instalar).
- Usuario no debería poder cambiar el servidor desde la UI.
- Instalador con nombre personalizado.
- Sin modificar código Rust si es posible (solo config/assets/build vars).

### Entregable 2 (alta prioridad) — Fix TCP Keepalive
- Aplicar keepalive TCP agresivo en el socket que cruza CGNAT (sesión cliente↔relay).
- Validación: conexión Postgres sobrevive >1 minuto idle a través del túnel.
- Issues upstream relacionados: **#11355**, **#487**, **#12431** (todos abiertos, ninguno mergeado).

### Entregable 3 (media prioridad) — Documentación
- Mantener este `NOTES.md` actualizado.
- Documentar comandos de rebuild local + CI.
- Documentar cómo rebasar contra futuros tags del upstream.

---

## 3. Estado del fork

| Campo | Valor |
| --- | --- |
| Branch de trabajo | `claude/rustdesk-fork-customization-Lxtku` |
| Último commit base upstream | `383a5c3` — `feat: option, enable-privacy-mode & enable-perm-change-in-accept-window (#14875)` |
| Versión upstream actual | `1.4.6` (`Cargo.toml`, `flutter/pubspec.yaml`) |
| Tags locales | Ninguno |
| Submodule `libs/hbb_common` | **Inicializado**. Apunta al fork `juliorgf/hbb_common` en SHA `a78a7f2` (branch `claude/rustdesk-fork-customization-Lxtku`). Carga el patch de SO_KEEPALIVE. |
| Cambios propios commiteados (parent) | `7a83078` (NOTES.md), `9aae148` (wiring del submodule fork), `0481bd1` (NOTES.md update post-fix). |
| Upstream remote git | No configurado en el repo local. URL upstream conocida: `https://github.com/rustdesk/rustdesk`. |
| Fork de `hbb_common` | `https://github.com/juliorgf/hbb_common` (creado por el usuario; push directo desde la sesión de Claude está bloqueado por el harness, el usuario lo pushea manualmente desde su máquina). |

### Decisión pendiente sobre versionado
- Hoy estamos parados sobre un commit de master upstream, no sobre un tag estable.
- Recomendación a discutir con el usuario: rebasar a un **tag estable** (ej. cuando se publique `1.4.7`) en lugar de seguir master.

---

## 4. Mapa del repositorio (referencia rápida)

Descripción detallada del layout en `AGENTS.md`. Resumen para nuestro trabajo:

| Carpeta | Para qué sirve en este fork |
| --- | --- |
| `src/` | Código Rust del cliente y servidor. Tocaremos archivos de tunneling acá. |
| `src/server/` | Lógica del lado server: clipboard/audio/video/network y connection state machine. |
| `src/port_forward.rs` | TCP tunneling — listener local (lado cliente). |
| `src/server/connection.rs` | Connection state machine — connect al destino del tunnel (lado server). |
| `src/custom_server.rs` | Decodifica config de servidor desde el nombre del .exe (mecanismo upstream). |
| `flutter/` | UI Flutter (multiplataforma). Strings y assets de UI. |
| `flutter/windows/runner/Runner.rc` | Metadata embebida en el .exe Windows (nombre, descripción, copyright). |
| `libs/hbb_common/` | **Submodule** con `config.rs` (defaults de `RENDEZVOUS_SERVERS`, `RS_PUB_KEY`, etc.). Hoy está vacío. |
| `res/` | Iconos `.ico`/`.png`/`.svg`, configs MSI, recursos de instalador. |
| `res/msi/preprocess.py` | Genera el MSI con flags `--app-name`, `--manufacturer`, `--version`. |
| `.github/workflows/flutter-build.yml` | Workflow principal que compila Windows/macOS/Linux/Android. |
| `build.py` | Orquestador Python de los builds nativos. |
| `build.rs` | Build script Rust — embebe ícono en el .exe Windows (`res.set_icon("res/icon.ico")`). |
| `Cargo.toml` | Workspace, version `1.4.6`, metadata winres y bundle. |

---

## 5. Puntos de rebranding identificados

### 5.1 Nombre del producto

| Archivo | Línea / sección | Qué define |
| --- | --- | --- |
| `Cargo.toml` | `[package.metadata.winres]` | `ProductName = "RustDesk"`, `FileDescription`, `OriginalFilename = "rustdesk.exe"` (embebidos en el .exe Windows). |
| `Cargo.toml` | `[package.metadata.bundle]` | `name = "RustDesk"`, `identifier = "com.carriez.rustdesk"` (bundle macOS). |
| `Cargo.toml` | `default-run = "rustdesk"` | Nombre del binario por defecto. |
| `flutter/pubspec.yaml` | `name: flutter_hbb` | Nombre interno Flutter. **No cambiar** (rompe imports). Versión `1.4.6+64`. |
| `flutter/windows/runner/Runner.rc` | ~líneas 92–98 | `CompanyName = "Purslane Ltd"`, `ProductName = "RustDesk"`, `FileDescription`, `OriginalFilename`. |
| `src/common.rs` | ~línea 56 | `PORTABLE_APPNAME_RUNTIME_ENV_KEY = "RUSTDESK_APPNAME"` — variable de entorno para override en runtime. |
| `src/common.rs` | `get_app_name()` | Lee `hbb_common::config::APP_NAME` (definido en submodule). |
| `build.py` | ~línea 17 | `hbb_name = 'rustdesk' + ('.exe' if windows else '')`. |
| `res/msi/preprocess.py` | ~líneas 76–89 | Defaults `--app-name "RustDesk"`, `--manufacturer "PURSLANE"`. |

**Estrategia recomendada para rebranding del nombre**:
- Cambiar `Cargo.toml` (winres + bundle) → afecta metadata del .exe.
- Cambiar `flutter/windows/runner/Runner.rc` → afecta título de ventana Flutter.
- Pasar `--app-name "Soporte INFOCOOP" --manufacturer "INFOCOOP"` a `preprocess.py` desde el workflow.
- **No** tocar `flutter/pubspec.yaml:name` (es un identificador interno del paquete Flutter, romper imports).
- Decidir nombre del binario: `soporte-infocoop.exe` vs mantener `rustdesk.exe` para minimizar diff.

### 5.2 Logos / iconos

| Path | Tamaño | Uso |
| --- | --- | --- |
| `res/icon.ico` | 99 KB | Ícono principal del .exe Windows (embebido vía `build.rs`). |
| `res/tray-icon.ico` | 4.3 KB | Ícono del tray. |
| `res/icon.png` | 40 KB | Ícono general. |
| `res/32x32.png`, `res/64x64.png`, `res/128x128.png`, `res/128x128@2x.png` | varios | Multi-resolución Linux/macOS. |
| `res/logo.svg`, `res/logo-header.svg`, `res/rustdesk-banner.svg`, `res/scalable.svg` | SVG | Branding/marketing. |
| `res/mac-icon.png`, `res/mac-tray-light-x2.png`, `res/mac-tray-dark-x2.png` | PNG | macOS. |
| `flutter/assets/*.svg` | varios | Iconos UI (no son el logo del producto, sí elementos como home/folder/etc — probablemente no tocar). |
| `flutter/windows/runner/resources/app_icon.ico` | ICO | Ícono del runner Flutter Windows (referenciado en `Runner.rc:55`). |

**Estrategia recomendada para logos**: reemplazar in-place los `.ico`/`.png`/`.svg` de `res/` con assets INFOCOOP, manteniendo nombres de archivo originales para no tocar paths.

### 5.3 Servidores por defecto (hbbs/hbbr/pubkey)

**Punto crítico**: las constantes `RENDEZVOUS_SERVERS` y `RS_PUB_KEY` viven en `libs/hbb_common/src/config.rs` (submodule no inicializado).

Mecanismos posibles para hardcodear servidores INFOCOOP, ordenados de menos a más invasivo:

1. **Custom server filename encoding** (mecanismo oficial upstream — `src/custom_server.rs`):
   - Renombrar el .exe instalado a:
     `rustdesk-host=remote.infocoop.com.py,key=Ly4NYMM5...,relay=remote.infocoop.com.py.exe`
   - El propio binario decodifica los parámetros del nombre del archivo al arrancar.
   - **Ventaja**: cero modificación de código Rust, máxima compatibilidad con futuros merges upstream.
   - **Desventaja**: si el usuario renombra el .exe, pierde la config.
2. **Modificar defaults en `libs/hbb_common/src/config.rs`** — requiere inicializar el submodule y commitear cambios. Más invasivo, mejor control, pero el submodule pertenece a otro repo (`rustdesk/hbb_common`) — habría que forkearlo también o aplicar override.
3. **Build-time env vars con `option_env!`** — requiere agregar lectura en `build.rs` o en código Rust. No existe hoy ese mecanismo en el repo. Más invasivo.

**Decisión preliminar a validar con el usuario**: opción 1 (filename encoding) como base, complementada con cambios en `flutter/windows/runner/Runner.rc` y `Cargo.toml` para metadata/branding. Si no alcanza, escalar a opción 2.

### 5.4 Bloquear cambio de servidor desde la UI

- Pendiente: identificar el screen de Flutter donde se editan los servers (probable: `flutter/lib/desktop/pages/connection_page.dart` o un settings screen).
- Mecanismo upstream: hay opciones tipo `OPTION_CUSTOM_RENDEZVOUS_SERVER` que pueden estar marcadas como "lockable" desde una build flag.
- **A investigar en Fase 1** cuando lleguemos al rebranding.

---

## 6. TCP Tunneling — análisis para el bug de keepalive

### 6.1 Diagnóstico (validado por el usuario)

- Síntoma: conexión Postgres a través del TCP tunnel muere a los ~30s idle.
- Causa raíz: CGNAT del ISP del operador mata conexiones idle a los 30s.
- `conntrack` del relay descartado (timeout 5 días).
- Keepalives de libpq (`keepalives_idle=10`) configurados y verificados — **no atraviesan el túnel** porque son per-socket en loopback (127.0.0.1) y nunca llegan al socket que cruza CGNAT.
- Fix conceptual: aplicar `set_tcp_keepalive()` con valores agresivos (idle ~15s, interval ~5s) en el socket que cruza CGNAT.

### 6.2 Sockets del flujo de tunneling

Topología real:

```
[psql/ODBC]
  ↓ (loopback 127.0.0.1)
[Cliente RustDesk] ── socket de SESIÓN RustDesk ──> [hbbr relay]
                       (este atraviesa CGNAT)              │
                                                  socket de SESIÓN RustDesk
                                                           ↓
                                              [Server RustDesk]
                                                           ↓ (loopback)
                                              [Postgres real :5432]
```

### 6.3 Sockets identificados en el código (hoy)

| Path | Línea | Qué socket es | ¿Cruza CGNAT? |
| --- | --- | --- | --- |
| `src/port_forward.rs` | 57 | `listener` en `127.0.0.1:{port}` | No (loopback). |
| `src/port_forward.rs` | 67 | `forward` (TcpStream aceptado en el listener) | **No (loopback).** Está en la conversación cliente local ↔ psql. |
| `src/port_forward.rs` | 72 | `Framed::new(forward, BytesCodec::new())` | wrapper sobre el loopback. |
| `src/port_forward.rs` | 73 | `stream` (devuelto por `connect_and_login`) | **SÍ — este es el `hbb_common::Stream` que va al relay.** |
| `src/server/connection.rs` | 1424 | `TcpStream::connect(&addr)` | **No (loopback)** — server conecta a Postgres real en localhost. |
| `src/server/connection.rs` | 1426 | `port_forward_socket = Some(Framed::new(sock, BytesCodec::new()))` | wrapper sobre loopback. |

### 6.4 Resolución (fix aplicado)

Tras inicializar el submodule y rastrear el flujo, el punto crítico resultó ser **`libs/hbb_common/src/tcp.rs:96-110`** (función `FramedStream::new`):

- Es el **único punto** donde se crea un `TcpStream` para conexiones outbound de RustDesk: cliente→relay, server→relay, cliente→rendezvous (hbbs), y todos los proxy paths terminan también acá.
- Ya tenía `set_nodelay(true)` aplicado al `stream` recién conectado (línea 99) — el `set_keepalive` va al lado, simétrico.
- No hay ningún uso de keepalive en TODO el repo (verificado por grep). RustDesk simplemente nunca seteó keepalive.

Patch aplicado en `juliorgf/hbb_common@a78a7f2`: agrega ~28 líneas justo después del `set_nodelay`, usando el mismo idiom raw-fd round-trip que ya usa `listen_any` en la misma file (líneas 234-247) para aplicar opciones de `socket2` a un `TcpSocket` de tokio. Soporta Unix y Windows. **Cero dependencias nuevas** (`socket2 0.3` ya estaba en `hbb_common/Cargo.toml`).

El submodule pointer está bumpeado en commit `9aae148` del parent.

### 6.5 Valores de keepalive elegidos

- `keepalive_time` (idle antes del primer probe): **15s** (vía `socket2::Socket::set_keepalive(Some(Duration::from_secs(15)))`).
- `keepalive_interval` y `keepalive_retries`: **defaults del OS**.
  - En Windows (la plataforma del cliente): interval ~1s, retries ~10 → probes activos por ~25s después de los primeros 15s idle. Esto mantiene la conexión "viva" desde la perspectiva del NAT, evitando el timeout de 30s del CGNAT.
  - En Linux (server / relay): interval 75s, retries 9 — en el server local del operador da igual, no cruza CGNAT.

Si el test real demuestra que los defaults son insuficientes en alguna plataforma, escalamos a `socket2 0.5` que expone `TcpKeepalive::with_interval(...).with_retries(...)`. Esto requiere bump de dependencia y consulta previa.

### 6.6 Valores propuestos originales (referencia histórica)

(Antes de implementar habíamos planeado controlar interval/retries con `socket2 0.5`. Se descartó por simplicidad — los defaults de Windows son suficientes para CGNAT 30s. Conservamos el plan por si el test real lo desmiente):

- `keepalive_time` (idle antes del primer probe): **15s**
- `keepalive_interval` (entre probes): **5s**
- `keepalive_retries`: **3**

Justificación: CGNAT mata a los 30s, queremos al menos un probe + ack antes de ese umbral, con margen para retransmisión.

### 6.6 Issues upstream a leer antes de patchar

- https://github.com/rustdesk/rustdesk/issues/11355
- https://github.com/rustdesk/rustdesk/issues/487
- https://github.com/rustdesk/rustdesk/issues/12431

(Pendiente: WebFetch en Fase 2 para ver si ya hay patches parciales propuestos.)

---

## 7. CI / GitHub Actions

### 7.1 Workflow principal

**Archivo**: `.github/workflows/flutter-build.yml`

- **Trigger**: `workflow_call` (reusable, invocado por `flutter-nightly.yml` y `flutter-tag.yml`).
- **Runner Windows**: `windows-2022`.
- **Toolchain**:
  - Rust `1.75` target `x86_64-pc-windows-msvc`
  - Flutter `3.24.5`
  - LLVM `15.0.6`
  - vcpkg commit pinned `120deac3...`
- **Comando de build principal** (línea ~167):
  `python3 .\build.py --portable --hwcodec --flutter --vram --skip-portable-pack`
- **Build MSI** (líneas ~258–266):
  ```bash
  pushd ./res/msi
  python preprocess.py --arp -d ../../rustdesk
  msbuild msi.sln -p:Configuration=Release -p:Platform=x64 /p:TargetVersion=Windows10
  ```

### 7.2 Secrets disponibles hoy

| Secret | Uso actual |
| --- | --- |
| `ANDROID_SIGNING_KEY` | Firma APK Android |
| `MACOS_P12_BASE64` / `MACOS_P12_PASSWORD` / `MACOS_CODESIGN_IDENTITY` | Firma macOS |
| `SIGN_BASE_URL` / `SIGN_SECRET_KEY` | Servicio de firma Windows externo |

**No existen secrets** para servidores INFOCOOP — habría que crear `INFOCOOP_RENDEZVOUS_SERVER` y `INFOCOOP_PUB_KEY` si vamos por la ruta build-time env vars.

### 7.3 Otros workflows

- `flutter-ci.yml`, `flutter-nightly.yml`, `flutter-tag.yml`, `bridge.yml`, `playground.yml`, `fdroid.yml`, `clear-cache.yml`, `ci.yml`.
- Vamos a tocar **solo `flutter-build.yml`** para inyectar branding/servers, o crear un workflow nuevo `infocoop-build.yml` específico para no contaminar los originales (mejor para mantenibilidad).

---

## 8. Acuerdos de trabajo

- **Roles**: usuario decide prioridades + valida + testea; Claude explora + propone + implementa + explica.
- **Idioma**: conversación en español, código y commits en inglés.
- **Commits**: Conventional Commits (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `build:`, `ci:`).
- **Branch**: todo en `claude/rustdesk-fork-customization-Lxtku`. No push a otra branch sin permiso explícito.
- **Consultas obligatorias** antes de:
  - Modificar más de 5 archivos en un solo cambio.
  - Agregar dependencias nuevas (Cargo.toml o pubspec.yaml).
  - Cambios fuera del scope de la fase actual.
  - Operaciones destructivas en git (rebase sobre commits ya pusheados, force-push, reset --hard).
- **Workflow por cambio**: propongo → implemento → muestro diff → commit → actualizo `NOTES.md` si corresponde.
- **Licencia**: AGPL-3.0 — mantener avisos de copyright y atribución a RustDesk upstream intactos.

---

## 9. Decisiones tomadas

| # | Decisión | Razón | Fecha |
| --- | --- | --- | --- |
| 1 | Trabajar sobre branch `claude/rustdesk-fork-customization-Lxtku`. | Branch designada por el usuario. | 2026-05-01 |
| 2 | Empezar por rebranding (Fase 1) antes que el fix de keepalive (Fase 2). | El rebranding tiene cero riesgo de romper funcionalidad y deja un build usable rápido. | 2026-05-01 |
| 3 | Documentar todo en `NOTES.md` como memoria entre sesiones. | Claude no tiene memoria persistente. | 2026-05-01 |
| 4 | Issue de keepalive: el socket crítico es el de la sesión RustDesk hacia el relay, NO los TcpStream de loopback identificados por la búsqueda inicial. | Análisis topológico: loopback no cruza CGNAT. Pendiente de confirmar exact path en el código en Fase 2. | 2026-05-01 |
| 5 | Reordenar: hacer Fase 2 (keepalive) ANTES que Fase 1 (rebranding). | El bug bloquea trabajo diario del usuario; el rebranding es estético. | 2026-05-02 |
| 6 | Fix de keepalive: aplicar en `libs/hbb_common/src/tcp.rs:99` (después de `set_nodelay`) en `FramedStream::new`. | Único punto donde se crea TcpStream para todas las conexiones outbound; `set_nodelay` ya está ahí, simétrico. | 2026-05-02 |
| 7 | Mantener `socket2 = "0.3"` (no upgradear). | El método `set_keepalive(Some(d))` en 0.3 alcanza para Windows (defaults del OS son agresivos suficiente para CGNAT 30s). Evitamos bump de dep. | 2026-05-02 |
| 8 | Forkear `hbb_common` a `juliorgf/hbb_common` y apuntar el submodule allí. | Es la única forma de tener nuestro patch persistido. Vendoring fue descartado por diff gigante. | 2026-05-02 |
| 9 | Push de `juliorgf/hbb_common` lo hace el usuario manualmente desde su máquina. | El harness de Claude tiene autorización solo para `juliorgf/rustdesk-infocoop`; intento de push a otro repo da `repository not authorized` 502. | 2026-05-02 |
| 10 | Strategy de servers preconfigurados: filename encoding (`src/custom_server.rs`). | Mecanismo upstream oficial, cero modificación de código Rust, máxima compatibilidad con futuros merges. | 2026-05-02 |
| 11 | Repo `juliorgf/rustdesk-infocoop` es **privado**. | Permite commitear datos del servidor en workflows sin necesidad de GitHub Secrets. | 2026-05-02 |
| 12 | Mantener `rustdesk.exe` como nombre del binario en lugar de `soporte-infocoop.exe`. | Minimizar diff vs upstream; la metadata visible al usuario igual va a decir "Soporte INFOCOOP" via `ProductName`/`FileDescription`. | 2026-05-02 |
| 13 | El workflow `infocoop-test-build.yml` se commitea en `master` (además de la feature branch). | GitHub solo muestra workflows con `workflow_dispatch` en el UI si están en la branch default. Push autorizado por el usuario para esta excepción puntual. | 2026-05-02 |

---

## 10. Cómo retomar en una sesión nueva (instrucciones para Claude)

1. Leer este `NOTES.md` completo.
2. Leer `AGENTS.md` (reglas de Rust/Tokio del proyecto).
3. Verificar estado git:
   ```
   git status
   git log --oneline -10
   git branch --show-current   # debe ser claude/rustdesk-fork-customization-Lxtku
   ```
4. Mirar la sección **§11 Próximos pasos** abajo para saber qué falta.
5. Confirmar con el usuario antes de avanzar.

---

## 11. Próximos pasos

### Fase 0 — Reconocimiento (terminada)
- [x] Mapear estructura del repo.
- [x] Identificar puntos de rebranding.
- [x] Identificar archivos relevantes para keepalive.
- [x] Identificar workflows de CI.
- [x] Crear `NOTES.md`.

### Fase 2 — Fix keepalive (código terminado, falta build + test)
- [x] Inicializar submodule `hbb_common`.
- [x] Mapear creación de `TcpStream` cliente↔relay (resultado: `libs/hbb_common/src/tcp.rs:99`).
- [x] Leer issues upstream #11355, #487, #12431 (WebFetch). Confirmado bug, ningún patch upstream útil.
- [x] Implementar `set_keepalive(Some(15s))` en el socket correcto. (commit `a78a7f2` en `juliorgf/hbb_common`).
- [x] Forkear `hbb_common` y wirear el submodule (commit `9aae148` en parent).
- [ ] **Build Windows con el fix aplicado**. Dos opciones:
  - **Recomendado**: GitHub Actions → workflow `INFOCOOP Test Build (Windows x64)` → Run workflow (ver §12). Genera un `.exe` unsigned descargable. ~30-60 min primera vez.
  - Local: requiere toolchain completa (Rust 1.75 + Flutter 3.24.5 + LLVM 15 + vcpkg) + `python3 build.py --portable --hwcodec --flutter --vram --skip-portable-pack`.
- [ ] **Test real contra CGNAT**: instalar el build, abrir TCP tunnel a Postgres, dejar idle >1 minuto, ejecutar query. Esperado: la conexión sobrevive.
- [ ] (Opcional) Si los defaults de Windows no alcanzan, escalar a `socket2 0.5` con `TcpKeepalive::with_interval(5s).with_retries(3)`.

### Fase 1 — Rebranding (después del fix validado)
- [ ] Logos: usuario subirá `infocoop.ico` (pendiente).
- [ ] Cambiar metadata winres/bundle en `Cargo.toml` (`ProductName`, `FileDescription`, `CompanyName`, `bundle.identifier`).
- [ ] Cambiar `flutter/windows/runner/Runner.rc` (`CompanyName`, `ProductName`, `FileDescription`).
- [ ] Reemplazar iconos en `res/icon.ico`, `res/tray-icon.ico`, `flutter/windows/runner/resources/app_icon.ico`, etc.
- [ ] Pasar `--app-name "Soporte INFOCOOP"` y `--manufacturer "INFOCOOP"` al MSI build (modificar workflow o defaults en `res/msi/preprocess.py`).
- [ ] Servidores preconfigurados: filename encoding via workflow CI (renombrar `rustdesk.exe` → `rustdesk-host=remote.infocoop.com.py,key=Ly4N...,relay=remote.infocoop.com.py.exe` post-build).
- [ ] (Opcional) Workflow CI dedicado `infocoop-build.yml` para no contaminar `flutter-build.yml`.

### Fase 3 — Documentación final
- [ ] Sección "How to rebuild" con comandos exactos (local + CI).
- [ ] Sección "How to rebase against new upstream tag".
- [ ] Sección "How to update hbb_common fork against upstream".
- [ ] Sección "Troubleshooting" para problemas comunes.

### Limpieza pendiente (decidir con el usuario)
- _(ninguna)_

---

## 12. Cómo disparar el build de testing en GitHub Actions

Workflow: `.github/workflows/infocoop-test-build.yml`. Trigger manual (`workflow_dispatch`).

**El archivo del workflow vive en `master`** (commit `c58ac03`), porque GitHub solo muestra workflows con `workflow_dispatch` en el UI si están en la branch default. Una copia idéntica también está en `claude/rustdesk-fork-customization-Lxtku` (commit `d06b673`). Mantener ambas en sync si se actualiza el workflow.

1. Entrar a https://github.com/juliorgf/rustdesk-infocoop/actions
2. Sidebar izquierdo: seleccionar **"INFOCOOP Test Build (Windows x64)"**.
3. Botón **"Run workflow"** (arriba a la derecha).
4. Branch: **seleccionar `claude/rustdesk-fork-customization-Lxtku`** (no master — el código con el fix está en la feature branch). El workflow definition se lee de master, pero el código a buildear sale de la branch elegida.
5. Confirmar **"Run workflow"**.

Tiempo esperado: 30-60 min la primera vez (vcpkg compila C++ deps), 15-30 min subsecuentes (con cache).

Output: artifact `rustdesk-windows-x64-keepalive-test` (ZIP con `rustdesk.exe` + DLLs). Disponible 14 días en la página del run.

**Limitaciones del build de testing**:
- Sin signing de Windows (SmartScreen puede dar warning al ejecutar — "Run anyway").
- Sin `RustDeskTempTopMostWindow` (componente nativo opcional, no afecta tunneling).
- Sin `usbmmidd_v2` ni printer driver (display virtual e impresoras — no afectan tunneling).
- Sin MSI ni portable packer — solo `.exe` + DLLs para ejecución directa.
- Sin servidores INFOCOOP preconfigurados — al primer arranque hay que configurar `remote.infocoop.com.py` + public key manualmente. Eso se automatiza en Fase 1.

---

## 13. Comandos útiles (referencia)

```bash
# Inicializar submodule hbb_common (necesario antes de compilar)
git submodule update --init --recursive

# Ver en qué commit base estamos parado vs upstream
git log --oneline -10

# Buscar usos de un símbolo en el código Rust
grep -rn "TcpStream::connect" src/ libs/

# Build local Windows (referencia, requiere toolchain completo)
python3 build.py --portable --hwcodec --flutter --vram --skip-portable-pack
```
