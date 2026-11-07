# OTClient Redemption Mobile — Roadmap (Fase 2)

Base: auditoria em `MOBILE_AUDIT.md`. Princípio: **reaproveitar o port GameActivity/EGL e os
módulos Lua existentes**; corrigir a plataforma antes de criar UI nova; nada de remover recurso desktop.

Itens marcados **[APROVAÇÃO]** são mudanças arquiteturais e só serão feitos após seu OK.

---

## 1. Arquitetura proposta

```
Android (Kotlin)                         C++ (NDK)                              Lua/OTUI
──────────────────                       ─────────────────────                  ─────────────────────────
MainActivity : GameActivity  ──JNI──▶    AndroidWindow (EGL, input)  ──InputEvent──▶ g_ui / UIWidget
AndroidManager (IME, clipboard,          AndroidManager (JNI bridge)            game_joystick (multitouch)
  densidade, safe-area, lifecycle)       ├─ Touch: pointerId → evento           game_shortcuts → game_mobile_hud
                                         │   por dedo (novo: TouchEvent)        game_mobile_hud (novo módulo):
                                         ├─ resize/close corretos                 ataque, alvo, magias, consumíveis
                                         └─ assets: extração versionada         estilos $mobile ampliados
```

Decisões:
1. **Manter GameActivity + EGL** (não migrar para SDL). O port já funciona com essa base; SDL
   implicaria reescrever `AndroidWindow` e o input, violando "não reescrever o que funciona".
2. **Multitouch [APROVAÇÃO]**: adicionar ao `InputEvent` um `touchId` e tipos
   `TouchPress/TouchMove/TouchRelease` (ou rotear cada ponteiro para o widget que recebeu o
   seu *down*). O ponteiro primário continua emulando mouse → todos os módulos atuais seguem
   funcionando; widgets "touch-aware" (joystick, botões do HUD) consomem os toques secundários.
   Afeta `framework/core/inputevent.h`, `ui/uimanager.cpp`, `ui/uiwidget*`, bindings Lua.
3. **HUD mobile como módulo Lua novo** (`modules/game_mobile_hud`) que *estende*
   `game_shortcuts`/`game_joystick` em vez de reescrevê-los, carregado só se `g_platform.isMobile()`.
4. **Hotkeys no mobile** reutilizam o modelo de dados do `game_actionbar` (slots com magia/item/
   texto), apenas com nova visualização touch — sem formato de configuração paralelo.
5. **Assets**: manter `data/things/<version>/` dentro do work dir (`game_data/`) como fonte única
   (contrato do AGENTS.md); `client_assets` continua sendo o instalador.

## 2. Arquivos a modificar (por etapa)

| Área | Arquivos |
|---|---|
| Lifecycle/resize/close | `src/framework/platform/androidwindow.cpp/.h`, `src/androidmain.cpp` |
| Input touch/teclado/gamepad | `androidwindow.cpp/.h`, `src/framework/core/inputevent.h`, `src/framework/ui/uimanager.cpp`, `src/framework/ui/uiwidget.cpp/.h`, `src/framework/luafunctions.cpp`, `android/.../FakeEditText.kt` |
| Assets | `src/framework/platform/androidmanager.cpp` (extração versionada/streaming), `android/app/build.gradle.kts` (injetar versão), workflow + `setup_android_deps.sh` (lista única de arquivos) |
| TLS | `src/framework/net/httplogin.cpp` (caminho do `cacert.pem` via resources; opção de verificação) |
| Safe-area/densidade | `android/.../MainActivity.kt`, `AndroidManager.kt`, `androidmanager.cpp`, `modules/game_interface` |
| HUD mobile | `modules/game_joystick/*`, `modules/game_shortcuts/*`, novo `modules/game_mobile_hud/*`, `modules/game_interface/gameinterface.lua`, `modules/client_options/*` |
| Inventário/containers | `modules/game_inventory`, `modules/game_containers` (apenas estilos `$mobile` + layout) |
| Scroll/zoom touch | `modules/corelib/ui/uiscrollarea.lua`, `modules/game_interface` (pinch → zoom) |
| Bugs menores | `modules/client_topmenu/topmenu.lua`, `mods/game_bot/executor.lua` |
| Build | `vcpkg.json` (filtrar `libarchive`, `discord-rpc` para `!android`), `Dockerfile.android` (fixar baseline), `android/BUILDING.md` |
| Manifest | remover permissões de storage não usadas; `configChanges` + `screenLayout|smallestScreenSize|density` |

## 3. Dependências necessárias

Nenhuma biblioteca nova é necessária para o primeiro APK. Ferramentas:
- JDK 17, Android SDK 36, NDK 29.0.13599879, CMake 3.22.1, vcpkg (baseline do `vcpkg.json`), gcc-multilib.
- Testes: emulador (AVD x86_64/arm64), `adb`, `logcat`; opcional AGDK Frame Pacing (*Swappy*) na fase de
  desempenho **[APROVAÇÃO]**.
- Assets Tibia de uma versão de teste (fornecidos por você; não são distribuídos no repositório).
- Um servidor de teste (ex.: Canary/TFS local) para login/reconexão.

## 4. Riscos técnicos

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Build vcpkg Android quebrar (ports/baseline) | média | alto | começar pelo CI existente; cache binário; fixar baseline no Docker |
| OOM ao extrair `data.zip` e/ou carregar sprites 13.x+ | média | alto | extração em streaming; testar 8.60 primeiro; medir com `dumpsys meminfo` |
| Mudança no `InputEvent` quebrar desktop | média | alto | tipos novos e aditivos; ponteiro primário continua mouse; teste manual desktop + `tests/` |
| Ciclo de vida EGL (background/foreground) causar crash/tela preta | alta | alto | corrigir resize/close; teste de 20 ciclos pause/resume |
| Threads do async dispatcher × JNI | baixa | médio | JNI só na thread principal; fila para chamadas vindas da UI thread Java |
| Login HTTPS falhar/inseguro | média | médio | localizar `cacert.pem` via work dir; verificação opcional configurável |
| Desempenho em GPUs low-end (Mali/Adreno antigas) | média | médio | medir FPS por resolução; limitar FPS/escala de render |
| Atualização do APK não atualizar Lua | alta | médio | marcador de versão na extração |

## 5. Etapas de implementação

Cada etapa = commits pequenos, build após cada um, critério de aceitação verificado antes da próxima.

**Etapa 0 — Baseline (sem alterar código)**
- Build do APK `arm64-v8a` via CI/Docker; instalar no emulador e device; registrar logcat.
- Resultado: relatório "o que funciona hoje" com evidências.

**Etapa 1 — Estabilidade de plataforma (mínimo para APK utilizável)**
1. Disparar `m_onResize` em `INIT_WINDOW`/`WINDOW_RESIZED`/`CONFIG_CHANGED` quando o tamanho mudar.
2. `destroyRequested` → `m_onClose`; não bloquear para sempre no `pollOnce`.
3. Tratar `ACTION_CANCEL`; resetar estado de drag por toque; consumir/limpar `keyEvents`.
4. Extração versionada do `data.zip` (versionCode), sem apagar `data/things/`, `data/sounds/` nem a pasta de escrita do usuário; extração em streaming.
5. Mapear teclas físicas básicas (setas, Esc, Tab, F1–F12, modificadores).
6. Filtrar `libarchive`/`discord-rpc` no `vcpkg.json` para `!android`; unificar lista do `data.zip`.

**Etapa 2 — Login e jogo**
- `cacert.pem` resolvido pelo work dir; teste de login 8.60 (TCP) e 13.x+ (HTTP login).
- Fluxo completo: login → personagens → entrar → andar → logout → relogar.

**Etapa 3 — Multitouch [APROVAÇÃO]**
- Eventos por dedo; joystick e botões simultâneos; pinch-zoom; arraste para rolar `UIScrollArea`.

**Etapa 4 — HUD mobile (Fase 4)**
- Joystick configurável (tamanho, opacidade, posição, zona morta, modo flutuante).
- Botões: atacar alvo mais próximo / próximo alvo, seguir, parar, usar, olhar.
- Barra de magias e consumíveis com slots configuráveis (dados do action bar).
- Menus por toque: long-press com feedback, menu radial opcional.
- Mapa ocupando o máximo (view mode 2 + painéis recolhíveis).
- Inventário e containers com estilos `$mobile` (slots maiores, janelas em tela cheia/aba).
- Safe-area/cutout; escala por densidade **e** resolução (dp mínimo por alvo de toque ≈ 48 dp).

**Etapa 5 — Qualidade (Fase 5)**
- Matriz de resoluções; FPS; memória; reconexão; latência; protocolos (ver §7).

## 6. Critérios de aceitação

| Funcionalidade | Critério |
|---|---|
| Inicialização | APK instala e chega à tela de login em < 10 s (2º boot) sem crash, em emulador e 1 device |
| Renderização | mapa e UI corretos; 20 ciclos background/foreground sem tela preta |
| Assets | atualizar APK atualiza módulos Lua; assets em `data/things/<v>/` preservados |
| Conexão | conecta em servidor 8.60 e 13.x+; erro claro em host inválido |
| Login/personagens | lista exibida; entra no jogo; logout/relogin funcionam |
| Movimentação | toque no mapa anda (autowalk); joystick anda nas 8 direções |
| Joystick | configurável e persistido; funciona junto com outro dedo |
| Combate | atacar/seguir/trocar alvo com 1 toque cada |
| Interface | nenhum alvo de toque < 40 dp; sem sobreposição com cutout |
| Inventário/containers | mover item por arraste; usar item; abrir/fechar containers |
| Hotkeys | criar, editar, remover slot; persiste após reiniciar |
| Desktop | build Linux/Windows inalterado e funcional |

## 7. Estratégia de testes

- **Build**: CI Android (arm64) + build desktop Linux em todo commit; erros de compilação reportados, nunca suprimidos.
- **Unit**: `tests/` (gtest) e `reusable-tests-lua.yml` para lógica não-gráfica (ex.: mapeamento de direção do joystick, versão de extração).
- **Instrumentado/manual** em emulador + device: roteiro reproduzível (boot, login, andar, combate, inventário, rotação, background).
- **Resoluções**: 1280×720 @2.0, 1920×1080 @2.75, 2400×1080 @3.0, tablet 2560×1600 @2.0.
- **Desempenho**: FPS do próprio cliente (`onFps`) + `adb shell dumpsys gfxinfo`; meta ≥ 30 FPS estável em device médio.
- **Memória**: `adb shell dumpsys meminfo com.github.otclient` em idle/cidade/combate; sem crescimento contínuo em 30 min.
- **Reconexão**: alternar Wi-Fi/dados e modo avião; app em background 5 min; verificar relogin.
- **Latência**: ping do cliente (`onPingBack`) comparado ao desktop na mesma rede.
- **Protocolos**: matriz mínima 8.60, 10.98, 13.x, 15.x (conforme servidores disponíveis).
- **Estabilidade**: sessão de 1 h sem crash; monitorar logcat por `FATAL`/ANR.

## 8. Próximo passo proposto

Executar a **Etapa 0** (build e teste do APK atual, sem alterar código). Para isso preciso de:
1. Confirmação de onde compilar (GitHub Actions do fork habilitado, ou máquina/Docker sua) — este
   container não tem Android SDK/NDK e o build vcpkg de todas as dependências é longo.
2. Versão de protocolo/servidor de teste e os respectivos assets.
3. Aprovação para seguir para a Etapa 1 após o relatório da Etapa 0.
