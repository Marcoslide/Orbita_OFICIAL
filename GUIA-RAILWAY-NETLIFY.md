# Órbita — Colocar em Produção (Railway + Netlify)

Guia direto para subir o **backend no Railway** e o **front no Netlify**, com
tudo ligado (Stripe, WhatsApp, Open Finance). Tempo estimado: 30–45 min.

> **Importante sobre "Stripe/WhatsApp reais":** o código já está 100% pronto para
> eles. O que você precisa fornecer são as **credenciais** (chaves), que só você
> gera nas contas da Stripe e da Meta. Sem as chaves, esses botões mostram uma
> mensagem clara ("Stripe não configurado", "Meta WhatsApp não configurada") em
> vez de quebrar. Tudo o mais (login, financeiro, IA, Open Finance sandbox, admin,
> limites de plano) funciona sem nenhuma credencial externa.

---

## Visão geral

```
   Você no navegador
        │
        ▼
  Netlify (front: index.html)  ──chama──►  Railway (API + Postgres + Redis)
```

- **Netlify**: hospeda só o `index.html` (a tela).
- **Railway**: roda a API NestJS, o Postgres e o Redis.

---

## PARTE 1 — Backend no Railway

### 1.1 Subir o código
Você precisa do código da API num repositório Git (GitHub). Se ainda não tem:
1. Crie um repositório no GitHub (pode ser privado).
2. Suba a pasta do projeto (a que contém `apps/api`, `railway.json`, etc.).
   O `orbita-producao-deploy.zip` já tem essa estrutura; descompacte e suba.

### 1.2 Criar o projeto no Railway
1. Acesse railway.app e faça login (pode usar o GitHub).
2. **New Project → Deploy from GitHub repo** → escolha seu repositório.
3. O Railway vai detectar o `railway.json` e usar o `apps/api/Dockerfile.railway`.
   - Se ele não detectar, em **Settings → Build**, aponte o Dockerfile para
     `apps/api/Dockerfile.railway`.

### 1.3 Adicionar Postgres e Redis
1. No projeto, clique **+ New → Database → Add PostgreSQL**.
2. Clique de novo **+ New → Database → Add Redis**.
3. O Railway cria as variáveis `DATABASE_URL` e `REDIS_URL` automaticamente.
   - Confirme que o serviço da API tem acesso a elas: em **Variables** da API,
     elas devem aparecer (se não, use "Add Reference" para linká-las).

### 1.4 Configurar as variáveis
No serviço da API, abra **Variables** e cole o conteúdo de
`RAILWAY-VARIAVEIS.env` (ajuste os valores). Mínimo para subir:
- `NODE_ENV=production`
- `JWT_SECRET` e `JWT_REFRESH_SECRET` → gere com `openssl rand -hex 32`
  (ou em qualquer gerador de hex de 64 caracteres).
- `WEB_ORIGIN` → deixe provisório, você ajusta depois de criar o site no Netlify.

As de Stripe/WhatsApp/OpenAI você cola quando tiver as chaves (Parte 3).

### 1.5 Deploy
1. O Railway faz o build e o deploy sozinho. Acompanhe em **Deployments**.
2. No primeiro boot, a API roda as migrations automaticamente
   (`prisma migrate deploy`) — isso cria todas as tabelas.
3. Quando ficar verde, vá em **Settings → Networking → Generate Domain**.
   Isso te dá a URL pública, algo como:
   `https://orbita-api-production.up.railway.app`

### 1.6 Testar a API
Abra no navegador:
```
https://SEU_BACKEND.up.railway.app/v1/health
```
Deve responder algo como `{"status":"ok","db":"up","redis":"up"}`. ✅

### 1.7 Criar o primeiro SuperAdmin
No Railway, abra o serviço da API → aba **Settings** (ou use a CLI do Railway) e
rode um comando único. Pela interface, o caminho mais simples:
1. Instale a CLI: `npm i -g @railway/cli` e faça `railway login`.
2. No terminal, dentro da pasta do projeto: `railway link` (escolha o projeto).
3. Rode:
   ```bash
   railway run --service API \
     bash -c 'ADMIN_EMAIL=voce@email.com ADMIN_PASSWORD=suaSenhaForte ADMIN_NAME="Seu Nome" npx ts-node prisma/seed-admin.ts'
   ```
   (troque `--service API` pelo nome do seu serviço, se for outro).

Pronto — esse e-mail/senha será seu login de SuperAdmin (vê a aba Admin SaaS).

---

## PARTE 2 — Front no Netlify

### 2.1 Apontar o front para a API
1. Abra o arquivo `index.html`.
2. Logo no topo do primeiro `<script>`, troque o placeholder:
   ```js
   window.ORBITA_API_URL = 'COLE_A_URL_DA_API_RAILWAY_AQUI';
   ```
   por:
   ```js
   window.ORBITA_API_URL = 'https://SEU_BACKEND.up.railway.app';
   ```
   (a URL que o Railway gerou no passo 1.5, **sem barra no final**).
3. Salve.

### 2.2 Subir no Netlify
1. Em app.netlify.com → **Add new site → Deploy manually**.
2. Arraste **o arquivo `index.html`** (ou uma pasta contendo só ele).
   - Não suba o zip da infra — só o `index.html`.
3. O Netlify te dá uma URL, ex.: `https://orbita-app.netlify.app`.

### 2.3 Liberar o CORS (passo que todo mundo esquece)
A API só aceita chamadas do domínio do front. Volte ao **Railway → Variables**
da API e ajuste:
```
WEB_ORIGIN=https://orbita-app.netlify.app
```
(a URL exata do seu site Netlify, sem barra no final). Salve — o Railway
reinicia a API sozinho. **Sem isso, o login falha com erro de CORS.**

### 2.4 Testar
1. Abra `https://orbita-app.netlify.app`.
2. Clique **Entrar** → deve abrir a **tela de login** (não o modo demo).
3. Entre com o e-mail/senha do SuperAdmin (passo 1.7).
4. Você cai no Dashboard real. A aba **Admin SaaS** deve aparecer. ✅

Se aparecer "API indisponível — Modo Local", revise: a URL no `index.html` está
certa? O `/v1/health` responde? O `WEB_ORIGIN` bate com a URL do Netlify?

---

## PARTE 3 — Ligar Stripe e WhatsApp (credenciais reais)

Faça depois que o login básico já estiver funcionando.

### 3.1 Stripe
1. Em dashboard.stripe.com (pode começar em **modo teste**):
   - **Developers → API keys**: copie a **Secret key** → `STRIPE_SECRET_KEY`.
   - **Products**: crie dois produtos com preço recorrente mensal (Pro, Business);
     copie os **Price IDs** → `STRIPE_PRICE_PRO` e `STRIPE_PRICE_BUSINESS`.
   - **Developers → Webhooks → Add endpoint**:
     - URL: `https://SEU_BACKEND.up.railway.app/v1/billing/webhook/stripe`
     - Eventos: `checkout.session.completed`, `customer.subscription.created`,
       `customer.subscription.updated`, `customer.subscription.deleted`,
       `invoice.payment_succeeded`, `invoice.payment_failed`,
       `customer.subscription.trial_will_end`.
     - Copie o **Signing secret** → `STRIPE_WEBHOOK_SECRET`.
2. Cole essas variáveis no Railway. A API reinicia.
3. No app, aba **Planos → Assinar Pro** → abre o checkout real da Stripe
   (cartão de teste: 4242 4242 4242 4242, validade futura, CVC qualquer).

### 3.2 WhatsApp (Meta Cloud API)
> Esta é a parte que mais demora, porque a Meta exige criar um App, um número e
> (para produção de verdade) verificação do negócio. Para testar, o número de
> teste que a Meta fornece já serve.
1. Em developers.facebook.com → crie um App do tipo Business → adicione **WhatsApp**.
2. Anote: **Phone number ID** → `WHATSAPP_PHONE_NUMBER_ID`; gere um
   **token de acesso** → `WHATSAPP_ACCESS_TOKEN`; em App Settings copie o
   **App Secret** → `WHATSAPP_APP_SECRET`.
3. Invente um texto qualquer para `META_VERIFY_TOKEN` (ex.: `orbita-webhook-123`).
4. Em **WhatsApp → Configuration → Webhook**:
   - Callback URL: `https://SEU_BACKEND.up.railway.app/v1/whatsapp/webhook`
   - Verify token: o mesmo `META_VERIFY_TOKEN`.
   - Assine o campo **messages**.
5. Cole as variáveis no Railway. A API reinicia.
6. No app, aba **WhatsApp → Conectar** (precisa de plano Pro/Business) gera o
   código `ORB-XXXXXX`.

### 3.3 OpenAI (opcional — só áudio/OCR)
- Em platform.openai.com → API keys → `OPENAI_API_KEY`. Sem ela, mensagens de
  texto funcionam; só transcrição de áudio e leitura de imagem ficam indisponíveis.

### 3.4 Pluggy (opcional — Open Finance real)
- Sem credenciais, o Open Finance roda em **sandbox** (banco de exemplo) e dá
  para testar todo o fluxo. Para bancos reais, preencha `PLUGGY_CLIENT_ID` /
  `PLUGGY_CLIENT_SECRET` e configure o webhook
  `https://SEU_BACKEND.up.railway.app/v1/open-finance/webhook/pluggy`.

---

## Checklist final

- [ ] `/v1/health` responde OK no Railway
- [ ] Migrations aplicadas (login não dá erro de tabela)
- [ ] SuperAdmin criado
- [ ] `index.html` aponta para a URL do Railway
- [ ] Site no Netlify abre e o botão Entrar mostra **login** (não demo)
- [ ] `WEB_ORIGIN` no Railway = URL do Netlify (CORS liberado)
- [ ] Login funciona, Dashboard carrega, aba Admin aparece
- [ ] (Stripe) checkout abre / mostra "não configurado" se sem chave
- [ ] (WhatsApp) gera ORB-XXXXXX / mostra "não configurada" se sem chave
- [ ] Open Finance sandbox conecta e importa

---

## Problemas comuns

| Sintoma | Causa provável | Solução |
|--------|----------------|---------|
| "Page not found" no Netlify | subiu o zip, não o `index.html` | suba só o `index.html` |
| "API indisponível — Modo Local" | URL errada ou `/health` fora do ar | confira `ORBITA_API_URL` e o deploy no Railway |
| Login dá erro de CORS | `WEB_ORIGIN` não bate | iguale ao domínio do Netlify, sem barra final |
| Erro de tabela no login | migrations não rodaram | veja os logs do Railway; o boot roda `migrate deploy` |
| Botão Assinar não faz nada | Stripe sem chave | normal — mostra "Stripe não configurado"; cole as chaves |
