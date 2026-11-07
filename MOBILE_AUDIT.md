# OTClient Redemption Mobile — Auditoria Técnica (Fase 1)

> Escopo: estado do port Android do OTClient (base `opentibiabr/otclient`, commit `c4822d9`).
> Método: leitura estática do código. **Nenhum build Android foi executado nesta auditoria**
> (o container usado não tem Android SDK/NDK; o fork não expõe execuções de GitHub Actions).
> Tudo marcado como "não verificado" precisa ser confirmado em build/dispositivo.

Legenda: ✅ pronto · 🟡 parcial/incompleto · ❌ ausente · ⚠️ defeito identificado

---

## 1. Arquitetura geral

```
┌──────────────────────── Lua (modules/, mods/) ────────────────────────┐
│ corelib → gamelib → client_* → game_* (interface, joystick, shortcuts) │
└───────────────▲───────────────────────────────────────────────────────┘
                │ bindings (luafunctions.cpp, luabinder)
┌───────────────┴──────────── C++ src/client ───────────────────────────┐
│ Game, ProtocolGame/Login, Map, MapView, Creature, Item, Things         │
└───────────────▲───────────────────────────────────────────────────────┘
┌───────────────┴──────────── C++ src/framework ────────────────────────┐
│ core (app, dispatcher, resources/PhysFS) · graphics (GL/GLES, drawpool)│
│ ui (UIWidget, OTML/OTUI, HTML/CSS) · net (asio, httplib, ixwebsocket)  │
│ sound (OpenAL) · platform (X11/Win32/Cocoa/Browser/**Android**)        │
└────────────────────────────────────────────────────────────────────────┘
```

- Build: CMake + vcpkg (manifest `vcpkg.json`). Alvo Android = `libotclient.so` (`add_library SHARED`).
- Toda a lógica de jogo e UI está em Lua/OTUI; o C++ fornece engine, rede, render e plataforma.
- Threads: thread principal (UI/Lua/eventos) + `g_asyncDispatcher` (BS thread pool) para o
  processamento de mapa/foreground (`graphicalapplication.cpp:197-240`).

## 2. Pasta `android/`

| Item | Estado | Observação |
|---|---|---|
| Gradle (AGP 8.12.3, Kotlin 1.9.23, compile/target SDK 36, minSdk 21) | ✅ | `android/app/build.gradle.kts` |
| NDK 29.0.13599879, CMake 3.22.1, STL `c++_shared` | ✅ | CMake raiz via `externalNativeBuild` |
| ABIs configuráveis (`OTCLIENT_ANDROID_ABIS`) | ✅ | default: 4 ABIs; CI usa só `arm64-v8a` |
| `MainActivity : GameActivity` (AGDK `games-activity:1.2.1`) | ✅ | **Não usa SDL** — usa GameActivity + EGL direto |
| `AndroidManager.kt` (teclado virtual, clipboard, densidade, preview de input) | ✅ | |
| `FakeEditText`/`NativeInputConnection` (IME → JNI) | 🟡 | só Enter/Backspace mapeados |
| Manifest: landscape fixo, GLES 3.0, INTERNET | 🟡 | pede `WRITE/READ_EXTERNAL_STORAGE` sem uso (obsoleto no API 33+) |
| Assinatura release | 🟡 | cai na debug keystore sem variáveis `RELEASE_*` |
| `isMinifyEnabled = false` | 🟡 | ok por ora (JNI depende de nomes de classes) |
| Assets: `data.zip` gerado por script e extraído no 1º boot | 🟡 ⚠️ | ver §5 |
| Ícone/tema/strings | ✅ | genéricos |

## 3. Integração C++ ↔ Android NDK

Arquivos: `src/androidmain.cpp`, `src/framework/platform/android{window,manager,gameactivity}.{h,cpp}`.

**Pronto**
- `android_main()` → espera EGL → chama `main()` padrão (reaproveita todo o fluxo desktop).
- Contexto EGL/GLES3, recriação de surface em `APP_CMD_INIT_WINDOW`/`TERM_WINDOW` e em `EGL_BAD_SURFACE`.
- Densidade via JNI (`getDisplayDensity`), coordenadas de toque divididas por densidade.
- Teclado virtual exibido ao focar `UITextEdit` por toque (`uitextedit.cpp:1371`).
- Áudio OpenAL pausado/retomado em `onPause/onResume`.

**Defeitos/lacunas identificados (não verificados em dispositivo)**
- ⚠️ `APP_CMD_WINDOW_RESIZED/CONFIG_CHANGED` só chamam `queryGlSize()`; **nunca disparam `m_onResize`**
  → UI não se redimensiona após o boot (rotação, split-screen, cutout, barra de navegação).
- ⚠️ `destroyRequested` só destrói a surface; não chama `m_onClose` → o processo pode ficar vivo/
  travado em `ALooper_pollOnce(-1)` após o Activity ser destruído.
- ⚠️ Multitouch: `ACTION_MOVE` empurra eventos de **todos** os ponteiros como um único mouse;
  `POINTER_DOWN/UP` caem em `EVENT_UNDEFINED`. Resultado: usar joystick + botão ao mesmo tempo
  faz o "cursor" saltar entre dedos. **Bloqueador para controles de combate.**
- ⚠️ `AMOTION_EVENT_ACTION_CANCEL` (=3) é tratado como `TOUCH_LONGPRESS` e gera um release sem
  limpar estado de botão.
- ⚠️ `processFingerMotion` usa `static lastMousePos` (não reinicia entre toques → falso "drag").
- 🟡 Long-press = clique direito só no *release* após 500 ms sem arrastar; sem feedback visual.
- ❌ Sem pinch-zoom, sem scroll por arraste, sem roda do mouse emulada.
- ❌ Teclado físico/gamepad: apenas `KEYCODE_ENTER(66)` e `KEYCODE_DEL(67)` mapeados; setas,
  F-keys, Esc, Tab, modificadores e `AInputEvent` de gamepad são ignorados (`android_app_set_key_event_filter(NULL)` aceita, mas nada é processado em `keyEvents`).
- 🟡 Conversão de texto JNI é Latin-1 (caracteres fora de 0xFF viram `?`) — coerente com a política
  de encoding do cliente (`docs/string-encoding-policy.md`), mas limita chat.
- 🟡 `nativeSetAudioEnabled` é chamado na UI thread Java e mexe em `g_sounds` sem sincronização.
- 🟡 `AndroidManager::getJNIEnv()` chama `AttachCurrentThread` a cada chamada e nunca faz detach.

## 4. Dependências nativas

| Dependência | Android | Observação |
|---|---|---|
| SDL | ❌ não usada | **Não há SDL no projeto.** Recomendação: *não* introduzir (ver roadmap §1) |
| OpenGL ES 3 / EGL | ✅ | `-DOPENGL_ES=3`; shaders recebem prefixo de precisão (`shader.cpp:50`) |
| LuaJIT | ✅ (pré-compilado) | `build_luajit_android.sh`, commit `d0e8893`, `-DLUAJIT_UNWIND_EXTERNAL` |
| OpenAL-soft, ogg/vorbis | ✅ | via vcpkg |
| PhysFS, zlib, liblzma, protobuf, abseil, fmt, spdlog, pugixml, nlohmann-json, freetype, libpng(apng) | ✅ | via vcpkg |
| OpenSSL / cpp-httplib / asio / ixwebsocket | ✅ | rede/login HTTP |
| libarchive | ✅ excluída do link | `if(NOT ANDROID)` — conforme AGENTS.md; mas ainda listada sem filtro de plataforma em `vcpkg.json` (instalada, não linkada) |
| discord-rpc | ✅ desativada | idem: instalada pelo manifest, não usada |
| minizip vendorizado | ✅ | extração do `data.zip` |
| game-activity (prefab) | ✅ | |

## 5. Carregamento de assets e módulos Lua

- `main.cpp:121` → `g_androidManager.unZipAssetData()` extrai `assets/data.zip` para
  `<internalDataPath>/game_data/`; `discoverWorkDir("init.lua")` encontra `game_data/`.
- `init.lua` monta `data/`, `modules/`, `mods/`; `setupUserWriteDir` usa `<internalDataPath>/`.
- `ModuleManager` carrega `.otmod` normalmente (sem diferença por plataforma) ✅.
- ⚠️ **A extração é pulada se `game_data/init.lua` já existe** → após atualizar o APK, módulos/
  Lua antigos continuam em uso. Precisa de marcador de versão (preservando `data/things/<v>`
  já baixados — contrato do AGENTS.md).
- ⚠️ `data.zip` é lido inteiro para RAM (`AASSET_MODE_BUFFER` + `malloc`) — risco de OOM se
  sprites forem empacotados (13.x/15.x ≈ centenas de MB).
- 🟡 `data/things/` vem vazio (só README). Assets de Tibia precisam ser: (a) empacotados no APK,
  (b) baixados pelo `client_assets` (grava no work dir → `game_data/data/things/<v>/`, respeitando
  o contrato), ou (c) copiados manualmente. `.rar` não extrai no Android (sem libarchive).
- 🟡 `httplogin.cpp:383` usa `set_ca_cert_path("./cacert.pem")` relativo ao CWD (inválido no
  Android) e **desativa verificação de certificado** — funciona, mas é inseguro (também no desktop).

## 6. Joystick e controles touch existentes

| Recurso | Estado | Onde |
|---|---|---|
| Joystick virtual (8 direções, fixo canto inferior esquerdo, 175×150) | 🟡 | `modules/game_joystick` |
| Integração com andar | ✅ | `game_walk/walk.lua:272` |
| Botões de modo: attack / look / follow / use (toggle "próximo toque") | 🟡 | `modules/game_shortcuts` |
| Joystick à direita (opção `rightJoystick`) | ✅ | `client_options/data_options.lua:592` |
| Botões zoom in/out no topo | ✅ | `client_topmenu/topmenu.lua:127` |
| Botão de chat flutuante | ✅ | `game_console/console.lua:2530` |
| Long-press → menu de contexto | 🟡 | via clique direito emulado |

Limitações: joystick depende de `g_mouse.isPressed(MouseLeftButton)` (um único "mouse"
global) e de polling de 20 ms; não é configurável (tamanho/opacidade/posição livre/zona morta);
não há botão de ataque direto ("atacar alvo mais próximo"), nem barra de magias touch dedicada.

## 7. Interface e densidade

- ✅ Estado de estilo `$mobile` / `$!mobile` no OTUI (`uiwidget.cpp:1831`), usado em ~15 arquivos.
- ✅ `g_platform.isMobile()` em ~40 pontos de Lua (painéis laterais ocultos, view mode 1/2, opções).
- ✅ `hudScale` default 1.5× no mobile (`data_options.lua:452`), sobrepõe a densidade do sistema.
- ✅ Tamanho mínimo 640×360 no mobile (`startup.lua:13`).
- 🟡 Escala única global; widgets desktop (mini-windows, menus, scrollbars 12 px) continuam
  pequenos para dedo. Não há safe-area/cutout.
- ⚠️ Pequeno bug: `topmenu.lua` `terminate()` testa `zoomOutButton:isDestroyed()` ao destruir `zoomInButton`.
- ⚠️ `mods/game_bot/executor.lua:124` usa `g_app.isMobile`, que não existe (só `g_platform.isMobile`).

## 8. Recursos dependentes de teclado/mouse

- Modificadores: `g_keyboard.is*Pressed()` retorna sempre `false` no mobile → Ctrl+clique,
  Shift+clique (look), Alt e arrastar com Ctrl (mover quantidade) ficam inacessíveis, exceto onde
  há alternativa (count window, menus).
- Hotkeys/keybinds: painel de keybinds é **ocultado** no mobile (`keybins.lua:731`); hotkeys
  (`game_hotkeys`) e action bar (`game_actionbar`) só disparam por tecla ou clique.
- Roda do mouse: 33 handlers `onMouseWheel` (scroll de listas, zoom do mapa) sem equivalente touch;
  `UIScrollArea` não suporta arraste para rolar.
- Hover/tooltips: sem hover em touch.
- Arrastar-e-soltar (inventário/containers): funciona como press-move-release, mas o limiar de
  drag e a falta de multitouch tornam frágil.
- Clique do meio, clique duplo-botão (classic control): inexistentes.

## 9. Compatibilidade de protocolos

- Lista suportada em `modules/gamelib/game.lua` (7.4 → 15.25). A camada de protocolo é
  independente de plataforma — **mesmo código no Android** ✅.
- 12.x+/13.x+ usam login HTTP(S) (`httplogin.cpp`, OpenSSL) — compila no Android; precisa teste real.
- Assets "protobuf" (11.x+/13.x+ `appearances.dat`) e `.dat/.spr` antigos usam o mesmo loader.
- Riscos: tamanho dos assets modernos em memória (texture atlas) em GPUs móveis; nenhuma
  medição existente.

## 10. Problemas conhecidos de compilação Android

- Build depende de 3 passos fora do Gradle: LuaJIT pré-compilado, vcpkg para cada triplet, `data.zip`.
- `armeabi-v7a` usa triplet `arm-neon-android`; x86/armv7 requerem `gcc-multilib`.
- `--allow-unsupported` necessário no vcpkg; ports `libarchive`/`discord-rpc` são instalados
  à toa (tempo de CI; risco de quebra de port não usado).
- CI (`.github/workflows/reusable-build-android-apk.yml`) compila apenas `arm64-v8a` release;
  não roda testes nem instala/inicia o APK em emulador.
- `Dockerfile.android` clona vcpkg sem fixar baseline (não reprodutível) e usa `thyrlian/android-sdk:10.0`.
- `BUILDING.md` e o workflow divergem na lista de arquivos do `data.zip` (CI inclui `cacert.pem`, `config.ini`).

## Resumo: pronto × incompleto × ausente

| Pronto | Incompleto | Ausente |
|---|---|---|
| Build Gradle+CMake+vcpkg (arm64 em CI) | Ciclo de vida (resize/close) | Multitouch real |
| EGL/GLES3, shaders ES | Extração/atualização de assets | Pinch-zoom, scroll touch |
| Boot via `main()` comum, Lua/módulos | Joystick (fixo, não configurável) | Barra de magias/consumíveis touch |
| Rede/protocolos (mesmo código) | Botões de ação (modo toggle) | Hotkeys configuráveis no mobile |
| Teclado virtual + clipboard | Escala de UI (global) | Gamepad/teclado físico completo |
| Áudio OpenAL | Login HTTPS (sem verificação TLS) | Testes automatizados em device/emulador |
| Estilos `$mobile` | Inventário/containers (UI desktop) | Safe-area / cutout |

## Caminho mais seguro para o primeiro APK funcional

1. **Não alterar código** inicialmente: rodar o pipeline existente (workflow de CI ou Docker)
   para `arm64-v8a`, com `data.zip` + um conjunto de assets de uma versão pequena (ex.: 8.60
   `.dat/.spr`) para reduzir risco de memória.
2. Instalar em emulador arm64/x86_64 e em 1 device físico; coletar `logcat -s OTClientMobile`.
3. Validar: boot → tela de login → conexão → lista de personagens → entrar no jogo → andar com joystick.
4. Só então aplicar as correções mínimas de plataforma (Etapa 1 do roadmap), uma por commit.

Detalhes, etapas e critérios: ver `MOBILE_ROADMAP.md`.
