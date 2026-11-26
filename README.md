# yt-pub-lives2

![yt-pub-lives2 Banner](assets/banner.jpg)

Pipeline automatizado para cortar lives do YouTube em clips por topico e publicar em outro canal.

**Canal de origem** (lives): [ARCKHÉ TDS](https://www.youtube.com/@linnoff88-devx) (`UC2QbQDyPKuHk93dwo5iq3Sw`)
**Canal de destino** (clips): [ARCKHÉ TIA](https://www.youtube.com/@ARCKHÉTIA) (`UCavuQHkxBSAZbzRoOm6Gq4g`)

## Fluxo

```
YouTube (lives do canal origem) â†’ Transcricao â†’ Analise IA â†’ Corte (FFmpeg) â†’ Thumbnail (IA) â†’ Publicacao (canal destino)
```

1. **Sincroniza** lives do canal de origem via YouTube Data API
2. **Baixa transcricao** automatica (legendas do YouTube)
3. **Analisa topicos** com IA (Piramyd/Claude/OpenRouter API)
4. **Corta clips** com FFmpeg baseado nos timestamps
5. **Gera thumbnails** com IA (LLM + gerador de imagem) ou local
6. **Publica clips** no canal de destino com titulo, descricao, tags e thumbnail

## Estrutura

```
yt-pub-lives2/
â”œâ”€â”€ config/                    # Configuracao isolada do projeto
â”‚   â”œâ”€â”€ .env                   # Variaveis de ambiente (nao vai pro git)
â”‚   â”œâ”€â”€ client_secret.json     # Credenciais OAuth (nao vai pro git)
â”‚   â”œâ”€â”€ credentials.enc        # Tokens encriptados (nao vai pro git)
â”‚   â”œâ”€â”€ .encryption_key        # Chave AES-GCM (nao vai pro git)
â”‚   â”œâ”€â”€ prompt_cortes.txt      # Prompt IA para analise de topicos
â”‚   â”œâ”€â”€ prompt_pub.txt         # Prompt IA para refinar titulo/descricao
â”‚   â””â”€â”€ prompt_thumb.txt       # Prompt IA para gerar thumbnails
â”œâ”€â”€ data/
â”‚   â””â”€â”€ lives.db               # Banco SQLite local (nao vai pro git)
â”œâ”€â”€ dashboard/
â”‚   â”œâ”€â”€ server.py              # Backend API (Python HTTP server)
â”‚   â””â”€â”€ index.html             # Frontend SPA (vanilla JS)
â”œâ”€â”€ scripts/
â”‚   â”œâ”€â”€ yt-auth                # Autenticacao OAuth standalone
â”‚   â”œâ”€â”€ yt-clip                # Pipeline: transcricao â†’ analise â†’ corte
â”‚   â”œâ”€â”€ yt-publish             # Upload de video para YouTube
â”‚   â”œâ”€â”€ yt-thumbnail           # Gera thumbnails com IA
â”‚   â”œâ”€â”€ setup-db               # Cria banco SQLite (com --import migra do Sheets)
â”‚   â””â”€â”€ sync-instances         # Sync codigo para outras instancias
â”œâ”€â”€ systemd/
â”‚   â”œâ”€â”€ yt-dashboard.service   # Service systemd (porta 8091)
â”‚   â””â”€â”€ yt-scheduler.service   # Service systemd scheduler
â”œâ”€â”€ db.py                      # Modulo SQLite (CONFIG, LIVES, PUBLICADOS)
â”œâ”€â”€ scheduler.py               # Scheduler automatico
â”œâ”€â”€ docker-compose.yml         # Docker (porta 8091)
â”œâ”€â”€ Dockerfile
â”œâ”€â”€ requirements.txt
â”œâ”€â”€ setup.sh
â””â”€â”€ docs/
    â””â”€â”€ SETUP-CANAL-DESTINO.md # Documentacao completa do setup
```

## Requisitos

- Python 3.10+
- ffmpeg
- yt-dlp
- deno (runtime JS para yt-dlp)
- curl
- Pillow (thumbnails)

## Instalacao

```bash
git clone git@github.com:linnoff88-dev/yt-pub-lives2.git
cd yt-pub-lives2
bash setup.sh
```

### 1. Configuracao Google Cloud (por instancia)

Cada instancia precisa de um **projeto Google Cloud proprio** com as seguintes configuracoes:

1. Acesse [Google Cloud Console](https://console.cloud.google.com) e crie um projeto (ex: `yt-pub-lives6`)
2. Ative a API: **YouTube Data API v3**
   - Menu: APIs & Services â†’ Library â†’ YouTube Data API v3 â†’ Enable
3. Configure o **OAuth Consent Screen**:
   - Menu: APIs & Services â†’ OAuth consent screen
   - Tipo: **External**
   - App name: qualquer nome (ex: `yt-pub-lives6`)
   - Scopes: adicione `youtube`, `youtube.upload`
   - Modo: **Testing**
   - Test users: adicione o **email da conta Google dona do canal de destino**
4. Crie credenciais **OAuth 2.0**:
   - Menu: APIs & Services â†’ Credentials â†’ Create Credentials â†’ OAuth client ID
   - Tipo: **Desktop App**
   - Authorized redirect URIs: adicione `http://localhost:8888`
   - Anote o **CLIENT_ID** e **CLIENT_SECRET**
5. Crie uma **API Key**:
   - Menu: APIs & Services â†’ Credentials â†’ Create Credentials â†’ API Key
   - Anote a **API_KEY**
6. **Verifique o telefone** do canal de destino em https://www.youtube.com/verify
   - Necessario para upload de thumbnails personalizadas
   - Sem verificacao, o pipeline funciona mas thumbnails nao sao enviadas

### 2. Configuracao do projeto

Crie o arquivo `config/.env` com as credenciais do projeto Google Cloud:

```env
# Canal de ORIGEM (de onde vem as lives â€” mesmo para todas as instancias)
YOUTUBE_CHANNEL_ID=UC2QbQDyPKuHk93dwo5iq3Sw

# Canal de DESTINO (credenciais OAuth da conta dona do canal)
# Cada instancia tem seu proprio projeto Google Cloud
CLIENT_ID=seu-client-id.apps.googleusercontent.com
CLIENT_SECRET=GOCSPX-seu-secret
API_KEY=AIzaSy-sua-api-key
GCP_PROJECT=nome-do-projeto-gcloud

# Piramyd API (IA para analise de topicos e thumbnails)
PIRAMYD_API_KEY=sk-sua-chave

# Nome da instancia (aparece no dashboard)
INSTANCE_NAME=yt-pub-livesN
```

### 3. Autenticacao OAuth

O OAuth conecta o sistema ao canal de destino. Cada instancia precisa autenticar com a conta Google dona do canal.

```bash
GWS_CONFIG_DIR=/caminho/para/config python3 scripts/yt-auth
```

O script:
1. Gera um link de autenticacao do Google
2. Abre um servidor local na porta 8888 (aguardando callback)
3. Voce abre o link no browser e autoriza com a conta do canal de destino
4. O callback salva os tokens encriptados em `config/credentials.enc`

**Comandos por instancia:**
```bash
# Lives1 (ARCKHÉ TDS)
GWS_CONFIG_DIR=/home/nmaldaner/.config/gws python3 scripts/yt-auth

# Lives2 (ARCKHÉ TIA)
GWS_CONFIG_DIR=/home/nmaldaner/projetos/yt-pub-lives2/config python3 scripts/yt-auth

# Lives3 (ARCKHÉ TDS)
GWS_CONFIG_DIR=/home/nmaldaner/projetos/yt-pub-lives3/config python3 scripts/yt-auth

# Lives4 (ARCKHÉ Tec)
GWS_CONFIG_DIR=/home/nmaldaner/projetos/yt-pub-lives4/config python3 scripts/yt-auth

# Lives5 (ARCKHÉ PROMPTS)
GWS_CONFIG_DIR=/home/nmaldaner/projetos/yt-pub-lives5/config python3 scripts/yt-auth

# Lives6 (ARCKHÉ Robot)
GWS_CONFIG_DIR=/home/nmaldaner/projetos/yt-pub-lives6/config python3 scripts/yt-auth
```

**Troubleshooting:**
- "Access blocked": clique em **Avancado** â†’ **Ir para (nome do app) (nao seguro)** (normal para apps em modo Testing)
- "Access blocked: app has not completed verification": a conta usada nao esta como test user â€” adicione em **APIs & Services â†’ OAuth consent screen â†’ Test users**
- "Unable to connect localhost:8888": o script `yt-auth` ja terminou â€” rode novamente e abra o link **enquanto o script esta rodando**
- Multiplas contas no browser: use **aba anonima** ou adicione `&login_hint=email@gmail.com` ao link

**Re-autenticacao via Master Dashboard (porta 8090):**

O master dashboard gera a URL OAuth com `redirect_uri=http://localhost:8090/api/auth/callback`. Para que o fluxo funcione, esse URI precisa estar cadastrado no Google Cloud Console do projeto da instancia:

1. Acesse [Google Cloud Console](https://console.cloud.google.com) â†’ selecione o projeto da instancia
2. APIs & Services â†’ Credentials â†’ clique no OAuth Client ID da instancia
3. Em **Authorized redirect URIs**, adicione: `http://localhost:8090/api/auth/callback`
4. Salve e aguarde ~1 min para propagar
5. Tente re-autenticar novamente pelo master dashboard

> Sintoma: re-autenticacao pelo master dashboard falha mesmo apos autorizar no Google â€” enquanto outras instancias funcionam. Causa: o `CLIENT_ID` do `.env` nao tem o redirect do master dashboard cadastrado no GCP.

### 4. Banco de Dados (SQLite local)

O banco e criado automaticamente ao iniciar o scheduler ou dashboard. Para criar manualmente:

```bash
# Criar DB vazio
python3 scripts/setup-db

# Criar DB e importar dados do Google Sheets (migracao de instancias antigas)
python3 scripts/setup-db --import
```

O banco fica em `data/lives.db` com 3 tabelas: **config**, **lives**, **publicados**.

### 5. Setup de nova instancia (passo a passo completo)

Exemplo para criar **lives7** na porta **8097**:

```bash
# 1. Copiar estrutura do lives2
cp -r ~/projetos/yt-pub-lives2 ~/projetos/yt-pub-lives7
rm -rf ~/projetos/yt-pub-lives7/data ~/projetos/yt-pub-lives7/lives ~/projetos/yt-pub-lives7/config/.env ~/projetos/yt-pub-lives7/config/credentials.enc ~/projetos/yt-pub-lives7/config/.encryption_key

# 2. Criar .env com credenciais do projeto Google Cloud
cat > ~/projetos/yt-pub-lives7/config/.env << 'EOF'
YOUTUBE_CHANNEL_ID=UC2QbQDyPKuHk93dwo5iq3Sw
CLIENT_ID=seu-client-id.apps.googleusercontent.com
CLIENT_SECRET=GOCSPX-seu-secret
API_KEY=AIzaSy-sua-api-key
GCP_PROJECT=seu-projeto
PIRAMYD_API_KEY=sk-sua-chave
INSTANCE_NAME=yt-pub-lives7
EOF

# 3. Autenticar OAuth (com a conta do canal de destino)
GWS_CONFIG_DIR=~/projetos/yt-pub-lives7/config python3 ~/projetos/yt-pub-lives7/scripts/yt-auth

# 4. Corrigir service files (porta e paths)
sed -i 's|lives2|lives7|g; s|8091|8097|g' ~/projetos/yt-pub-lives7/systemd/yt-dashboard.service
sed -i 's|lives2|lives7|g; s|dashboard2|dashboard7|g' ~/projetos/yt-pub-lives7/systemd/yt-scheduler.service
# Remover User=nmaldaner dos services (user services nao precisam)
sed -i '/^User=/d' ~/projetos/yt-pub-lives7/systemd/*.service

# 5. Criar symlinks e habilitar
ln -sf ~/projetos/yt-pub-lives7/systemd/yt-scheduler.service ~/.config/systemd/user/yt-scheduler7.service
ln -sf ~/projetos/yt-pub-lives7/systemd/yt-dashboard.service ~/.config/systemd/user/yt-dashboard7.service
systemctl --user daemon-reload
systemctl --user enable --now yt-scheduler7 yt-dashboard7

# 6. Criar diretorio de lives
mkdir -p ~/projetos/yt-pub-lives7/lives

# 7. Adicionar ao sync-instances (editar scripts/sync-instances no lives2)
# Adicionar: "/home/nmaldaner/projetos/yt-pub-lives7" ao array TARGETS
```

Dashboard: `http://192.168.1.91:8097`

### 6. Prompts de IA (opcional)

Copie os prompts personalizados para `config/`:
```bash
cp ~/caminho/prompt_cortes.txt config/
cp ~/caminho/prompt_pub.txt config/
cp ~/caminho/prompt_thumb.txt config/
```

Ou edite pelo dashboard na aba de configuracao.

## Uso

### Dashboard Web

```bash
python3 dashboard/server.py [porta]    # padrao: 8091
```

Acesse `http://localhost:8091` â€” painel com:
- Stats clicaveis (total lives, cortadas, pendentes, clips aguardando, publicados)
- Configuracao de horarios (picker visual 24h)
- Tabela de lives com filtro por status
- Aba Clips unificada: publicados + pendentes
- Controle de clips: pausar/retomar publicacao individual
- Reprocessar lives com erro
- Controle de privacy
- Configuracao de thumbnails
- Status do scheduler em tempo real

### Docker

```bash
docker-compose up -d
```

Dashboard em `http://localhost:8091`.

### Systemd (user services)

```bash
# Criar symlinks (exemplo para lives5, porta 8095)
ln -sf /home/nmaldaner/projetos/yt-pub-lives5/systemd/yt-scheduler.service ~/.config/systemd/user/yt-scheduler5.service
ln -sf /home/nmaldaner/projetos/yt-pub-lives5/systemd/yt-dashboard.service ~/.config/systemd/user/yt-dashboard5.service
systemctl --user daemon-reload
systemctl --user enable --now yt-scheduler5 yt-dashboard5
```

### Multi-instancia

| Instancia | Porta | Scheduler | Dashboard | Canal Destino | GCP Project |
|-----------|-------|-----------|-----------|---------------|-------------|
| lives1 | 8091 | yt-scheduler1 | yt-dashboard1 | ARCKHÉ TDS | gws |
| lives2 | 8092 | yt-scheduler | yt-dashboard | ARCKHÉ TIA | webyt |
| lives3 | 8093 | yt-scheduler3 | yt-dashboard3 | ARCKHÉ TDS | yt-pub-lives3 |
| lives4 | 8094 | yt-scheduler4 | yt-dashboard4 | ARCKHÉ Tec | yt-pub4 |
| lives5 | 8095 | yt-scheduler5 | yt-dashboard5 | ARCKHÉ PROMPTS | prompts-491620 |
| lives6 | 8096 | yt-scheduler6 | yt-dashboard6 | ARCKHÉ Robot | ARCKHÉ-robot |

**Sync codigo** (lives2 e a fonte):
```bash
./scripts/sync-instances    # Copia codigo para lives1,3,4,5,6
```

**Restart todos:**
```bash
systemctl --user restart yt-scheduler1 yt-dashboard1 yt-scheduler yt-dashboard yt-scheduler3 yt-dashboard3 yt-scheduler4 yt-dashboard4 yt-scheduler5 yt-dashboard5 yt-scheduler6 yt-dashboard6
```

### Cortar uma Live

```bash
yt-clip <video_id>                    # Modo manual (gera prompt)
yt-clip <video_id> --ai piramyd-api   # Modo automatico (Piramyd API)
yt-clip <video_id> --dry-run          # So mostra topicos
yt-clip <video_id> --publish          # Corta e publica
```

### Gerar Thumbnail

```bash
yt-thumbnail --title "Titulo do clip" --output thumb.jpg
```

### Publicar um Video

```bash
yt-publish video.mp4 --title "Titulo" --description "Descricao"
yt-publish video.mp4 --title "Titulo" --description "Desc" --privacy unlisted --tags "ia,dev"
```

## Diferenca do projeto original (yt-pub-lives)

| | yt-pub-lives | yt-pub-lives2 |
|---|---|---|
| Config | Global (`~/.config/gws/`) | Local (`./config/`) |
| Porta | 8090 | 8091 |
| Canal destino | Mesmo da origem | Diferente (ARCKHÉ TIA) |
| Auth | Via CLI `gws` | Script `yt-auth` standalone |
| Repositorio | `linnoff88-dev/yt-pub-lives` | `linnoff88-dev/yt-pub-lives2` |

## Tecnologias

- **Backend**: Python 3 (stdlib HTTPServer, sem frameworks)
- **Frontend**: HTML/CSS/JS vanilla (single page, sem build)
- **Banco**: SQLite local (WAL mode, sem dependencia externa)
- **APIs**: YouTube Data API v3
- **IA**: Piramyd API / Anthropic Claude API / OpenRouter (analise de topicos + thumbnails)
- **Video**: FFmpeg (corte), yt-dlp (download)
- **Auth**: OAuth 2.0 com refresh token (AES-GCM encrypted)

## Licenca

Uso interno â€” ARCKHÉ TDS (@linnoff88-devx)

---

Projeto  - ARCKHÉe