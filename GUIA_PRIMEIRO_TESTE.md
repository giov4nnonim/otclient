# Guia do primeiro teste — servidor local + OTClient (Windows → Android)

Objetivo: ver um personagem andando, primeiro no PC e depois no celular, usando
**somente fontes oficiais** (servidor Canary, cliente oficial do Tibia) — sem baixar
assets de repositório de terceiros.

> Itens marcados com ⚠️ ainda **não foram verificados na prática**. Anote o que
> acontecer neles e me envie.

---

## Os 3 ingredientes

| Ingrediente | Onde vem | Onde fica |
|---|---|---|
| **Servidor** (Canary) | `github.com/opentibiabr/canary` (projeto oficial da mesma comunidade do OTClient) | Seu PC, via Docker |
| **Cliente** (OTClient) | Seu fork `giov4nnonim/otclient` (build do GitHub Actions) | Seu PC e depois o celular |
| **Assets** (desenhos/sons) | Cliente **oficial** do Tibia (tibia.com) | Dentro da pasta do OTClient |

Os três precisam estar na **mesma versão** (ex.: 15.11).

---

## Fase 1 — Tudo no PC (Windows)

### Passo 1 — Instalar o Docker Desktop
- Baixe em `docker.com` → *Docker Desktop for Windows* e instale (pode pedir para ativar o WSL2 e reiniciar).
- Abra o Docker Desktop e espere aparecer "Engine running".
- Por que Docker: o Canary oficial traz uma receita pronta que sobe **servidor + banco de dados + login** de uma vez. Sem compilar nada no Visual Studio.

### Passo 2 — Baixar o Canary
- Instale o **Git for Windows** (`git-scm.com`) se ainda não tiver.
- No PowerShell:
  ```powershell
  cd $HOME
  git clone --depth 1 https://github.com/opentibiabr/canary.git
  cd canary\docker
  copy .env.dist .env
  ```
- O arquivo `.env` guarda as configurações (portas, senhas do banco). Para o teste local, deixe como está.

### Passo 3 — Ligar o servidor
- Ainda em `canary\docker`, no PowerShell:
  ```powershell
  .\up.ps1
  ```
  (equivale a `docker compose up -d --build`). A primeira vez demora: baixa o banco, o servidor e o mapa.
- Segundo a documentação oficial (`docker/DOCKER.md`), ficam disponíveis:
  - Login do jogo (webservice): `http://localhost:8088/login`
  - Servidor: portas **7171** (login), **7172** (jogo), **7173** (status)
  - Site (opcional): `http://localhost:8080`
- **Conta de teste**: a documentação diz `@test1` / senha `test`, **se** `CANARY_TEST_ACCOUNTS=true` estiver no `.env`. Confira essa linha no `.env` antes de subir. ⚠️

### Passo 4 — Descobrir a versão e pegar os assets oficiais
- Baixe e instale o **cliente oficial do Tibia** em `tibia.com` e deixe-o atualizar.
- Anote a **versão** que ele mostra (ex.: 15.11 → no OTClient vira `1511`).
- **Confira a compatibilidade** ⚠️:
  - O OTClient deste projeto aceita até a versão **15.25** (`1525`).
  - A última versão do Canary (v3.6.1) indica o cliente **15.11** nas notas de lançamento, e a descrição do repositório fala em **15.25**.
  - Se o cliente oficial estiver numa versão **mais nova** do que o OTClient/Canary aceitam, me avise antes de seguir.
- Encontre a pasta `assets` do cliente oficial (a que contém `catalog-content.json`). Normalmente fica dentro da pasta de instalação do Tibia, em algo como `...\Tibia\packages\Tibia\assets\`. ⚠️ (o caminho exato pode variar; procure por `catalog-content.json` no Explorador).

### Passo 5 — Preparar o OTClient para Windows
- Baixe o artifact **`windows-cmake-release`** do GitHub Actions do seu fork e descompacte, por exemplo em `C:\OTClient\`.
- Abra `C:\OTClient\init.lua` num editor de texto e confirme que está assim (o build novo já vem assim; um build antigo não):
  ```lua
  clientAssets = {
      enabled = false,
  ```
  Isso **desliga o download automático** de terceiros.
- Copie **o conteúdo** da pasta `assets` oficial para:
  ```
  C:\OTClient\data\things\1511\
  ```
  (troque `1511` pela sua versão). Dentro dela deve aparecer `catalog-content.json` diretamente — não uma subpasta `assets`.

### Passo 6 — Entrar no jogo pelo PC
- Abra o `otclient.exe`.
- Na tela de login ⚠️ (o Docker do Canary usa login via web):
  - **Servidor**: `http://127.0.0.1:8088/login`
  - **Porta**: `8088`
  - **Versão**: a sua (ex.: `1511`)
  - Marque **HTTP login** (é `http`, não `https`)
  - **Conta**: `@test1` — **Senha**: `test`
- ✅ **Sucesso da Fase 1** = escolher o personagem e conseguir andar.
- Se falhar: me mande `C:\OTClient\otclient.log` (ou o arquivo de log da pasta) e a mensagem de erro.

---

## Fase 2 — Celular Android (mesma rede Wi-Fi)

### Passo 7 — Liberar o PC para o celular
- Descubra o IP do PC: PowerShell → `ipconfig` → "Endereço IPv4" (ex.: `192.168.0.10`).
- Permita no **Firewall do Windows** as portas **7171, 7172 e 8088** (TCP) para rede privada.
- ⚠️ O servidor precisa anunciar o IP da rede (não `127.0.0.1`) para o celular conseguir entrar no mundo. Isso é configuração do Canary (`ip` no config / `.env`); verificamos juntos quando chegar aqui.

### Passo 8 — Instalar o app e criar a pasta dele
- Instale o APK novo (precisa ser gerado **depois** destas mudanças — ver "Pendências").
- **Abra o app uma vez e feche.** Isso faz o Android criar a pasta do app.

### Passo 9 — Colocar os assets no celular
- Com o celular no USB e a "Depuração USB" ligada, no PowerShell (com `adb` instalado — Android Platform Tools):
  ```powershell
  adb shell mkdir -p /sdcard/Android/data/com.github.otclient/files/import/data/things
  adb push C:\OTClient\data\things\1511 /sdcard/Android/data/com.github.otclient/files/import/data/things/1511
  ```
- Ao abrir o app de novo, ele **copia** esses arquivos para a pasta interna dele e **apaga** a cópia temporária. A primeira abertura depois disso pode demorar (são centenas de MB).
- Conferir: `adb logcat | findstr "Import"` deve mostrar `Imported assets for '1511'`.

### Passo 10 — Entrar pelo celular
- Mesmos dados do Passo 6, trocando `127.0.0.1` pelo IP do PC (ex.: `http://192.168.0.10:8088/login`).
- ✅ **Sucesso da Fase 2** = andar com o joystick.
- Me envie: até onde chegou, prints e o log: `adb logcat -d > otc_log.txt`.

---

## Segurança — o que mudou

- **Download automático de assets desligado** (`init.lua`): o cliente não busca mais nada de repositório de terceiros no login.
- **Assets vêm do cliente oficial**, copiados por você.
- **Importação no Android** só aceita `data/things/<número>` e `data/sounds/<número>`; ignora qualquer outra pasta (ex.: scripts) e links simbólicos.
- Ainda dependemos de: imagens Docker do Canary (`ghcr.io/opentibiabr/...`, projeto oficial) e bibliotecas do build (com versões fixadas).

## Pendências

- [ ] Religar o **GitHub Actions** do fork para gerar os builds novos (Windows e APK) com estas mudanças. O código Android novo **ainda não foi compilado com o NDK** — só testado isoladamente no Linux.
- [ ] Confirmar versão do cliente oficial × OTClient × Canary (Passo 4).
- [ ] Confirmar o modo de login com o Docker do Canary (Passo 6).
