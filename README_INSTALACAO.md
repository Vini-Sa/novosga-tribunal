# Sistema de Triagem NovoSGA — Documentação de Instalação
**Versão do documento:** 1.1  
**Data:** 16/04/2026  
**Finalidade:** Instalação e configuração do sistema de triagem NovoSGA em ambiente de produção via Docker.

---

## Visão Geral do Sistema

O sistema é composto por dois projetos independentes que se comunicam via API:

```
┌─────────────────────────────────────────────────────────────┐
│                        SERVIDOR                             │
│                                                             │
│  ┌──────────────────────┐   ┌──────────────────────────┐   │
│  │   NovoSGA v2.3       │   │   Triage App v2.1.0      │   │
│  │   (Sistema principal)│   │   (Totem de triagem)     │   │
│  │                      │   │                          │   │
│  │  • Painel admin      │   │  • Tela para o cidadão   │   │
│  │  • Painel atendimento│   │  • Emissão de senhas     │   │
│  │  • Gestão de filas   │◄──│  • Seleciona serviço     │   │
│  │                      │   │                          │   │
│  │  Porta: 8000         │   │  Porta: 8080             │   │
│  └──────────────────────┘   └──────────────────────────┘   │
│           │                                                 │
│  ┌────────┴───────────┐   ┌──────────────────────────┐     │
│  │  PostgreSQL 16     │   │  Mercure Hub             │     │
│  │  (Banco de dados)  │   │  (Tempo real / SSE)      │     │
│  │  Porta: interna    │   │  Porta: 3000             │     │
│  └────────────────────┘   └──────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

| Componente | Tecnologia | Função |
|---|---|---|
| **NovoSGA v2.3** | PHP/Symfony + nginx | Sistema principal: admin, atendimento, filas |
| **Triage App v2.1.0** | Vue.js + nginx | Totem: interface do cidadão para emissão de senha |
| **PostgreSQL 16** | Banco de dados | Armazenamento de todos os dados do NovoSGA |
| **Mercure Hub** | Caddy/Mercure | Atualização em tempo real do painel de atendimento |

---

## Pré-requisitos

- **Docker Desktop** (Windows) ou **Docker Engine + Docker Compose** (Linux/servidor)
- **Git** (para clonar os repositórios)
- Portas **8000**, **8080** e **3000** livres no servidor

> Em produção no servidor Linux, substituir `localhost` pelo IP ou domínio do servidor em todas as configurações.

---

## Estrutura de Pastas

```
/opt/novosga/                    ← diretório raiz sugerido para o servidor
├── novosga-docker/              ← sistema principal (este repositório)
│   ├── novosga-2.3-standalone/  ← Dockerfile do NovoSGA (modificado)
│   │   └── Dockerfile
│   ├── docker-compose.yml       ← orquestração dos containers (criado)
│   └── ...
└── triage-app-2.1.0/            ← app de triagem (modificado)
    ├── src/
    │   ├── index.ejs            ← modificado
    │   └── renderer/pages/
    │       └── Settings.vue     ← modificado
    ├── .electron-vue/
    │   └── webpack.web.config.js ← modificado
    ├── package.json             ← modificado
    └── Dockerfile               ← modificado
```

---

## Parte 1 — NovoSGA (Sistema Principal)

### 1.1 Clonar o repositório

```bash
git clone https://github.com/novosga/docker novosga-docker
cd novosga-docker
```

### 1.2 Arquivo `docker-compose.yml` (criado do zero)

O repositório original inclui um `docker-compose.yml` para **v2.0 com MySQL**, que não é compatível com a v2.3. Esse arquivo deve ser **substituído** pelo abaixo na raiz do repositório clonado:

```yaml
services:

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: novosga
      POSTGRES_USER: novosga
      POSTGRES_PASSWORD: novosga@tribunal     # ← alterar em produção
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U novosga"]
      interval: 10s
      timeout: 5s
      retries: 5

  mercure:
    image: dunglas/mercure
    environment:
      SERVER_NAME: ':3000'
      MERCURE_PUBLISHER_JWT_KEY: 'novosga_mercure_secret_tribunal_2026'   # ← alterar em produção
      MERCURE_SUBSCRIBER_JWT_KEY: 'novosga_mercure_secret_tribunal_2026'  # ← alterar em produção
      MERCURE_EXTRA_DIRECTIVES: |
        cors_origins *
        anonymous
    ports:
      - "3000:3000"

  novosga:
    build: ./novosga-2.3-standalone
    ports:
      - "8000:8080"
    environment:
      DATABASE_URL: "postgresql://novosga:novosga@tribunal@postgres:5432/novosga?serverVersion=16"
      MERCURE_URL: "http://mercure:3000/.well-known/mercure"
      MERCURE_PUBLIC_URL: "http://localhost:3000/.well-known/mercure"   # ← trocar localhost pelo IP/domínio do servidor
      MERCURE_JWT_SECRET: "novosga_mercure_secret_tribunal_2026"        # ← igual às chaves do Mercure acima
      NOVOSGA_ADMIN_USERNAME: admin
      NOVOSGA_ADMIN_PASSWORD: "admin@tribunal"    # ← alterar em produção
      NOVOSGA_ADMIN_FIRSTNAME: Administrador
      NOVOSGA_ADMIN_LASTNAME: Global
      NOVOSGA_UNITY_NAME: "Tribunal de Justiça"   # ← nome da unidade
      NOVOSGA_UNITY_CODE: TJ01
      NOVOSGA_NOPRIORITY_NAME: Normal
      NOVOSGA_NOPRIORITY_DESCRIPTION: "Atendimento normal"
      NOVOSGA_PRIORITY_NAME: "Prioritário"
      NOVOSGA_PRIORITY_DESCRIPTION: "Atendimento prioritário"
      NOVOSGA_PLACE_NAME: "Guichê"
      TZ: America/Sao_Paulo
    depends_on:
      postgres:
        condition: service_healthy
      mercure:
        condition: service_started

volumes:
  postgres_data:
```

> **Atenção em produção:** substituir todos os valores marcados com `← alterar em produção`. O `MERCURE_JWT_SECRET` deve ter **no mínimo 32 caracteres** (256 bits) — requisito do algoritmo JWT HS256.

### 1.3 Modificação no Dockerfile do NovoSGA (`novosga-2.3-standalone/Dockerfile`)

**Motivo:** O repositório é clonado no Windows, que converte automaticamente os finais de linha dos arquivos `.sh` de LF (Linux) para CRLF (Windows). O Linux não consegue executar scripts com CRLF, gerando o erro `no such file or directory` mesmo o arquivo existindo.

**Alteração adicionada ao final do Dockerfile:**
```dockerfile
RUN sed -i 's/\r$//' /usr/local/bin/start.sh /usr/local/bin/consumer.sh && \
    chmod +x /usr/local/bin/start.sh /usr/local/bin/consumer.sh
```

> **Nota:** Em servidores Linux onde o git não converte finais de linha, esta linha é inofensiva — o `sed` simplesmente não encontra `\r` para remover.

### 1.4 Ajuste obrigatório antes de subir em produção

No arquivo `novosga-docker/docker-compose.yml`, alterar a variável:

```yaml
MERCURE_PUBLIC_URL: "http://[IP-ou-domínio-do-servidor]:3000/.well-known/mercure"
```

> Essa é a única variável que precisa ser editada diretamente no arquivo. Sem essa alteração, o painel de atendimento não atualiza em tempo real.

### 1.5 Subir o sistema

O `docker-compose.yml` orquestra todos os serviços de uma vez: PostgreSQL, Mercure, NovoSGA e Triage App.

```bash
cd novosga-docker
docker compose up -d
```

Na primeira execução, o sistema:
1. Baixa as imagens do Docker Hub e faz o build do NovoSGA e do Triage App
2. Aguarda o PostgreSQL ficar pronto (healthcheck)
3. Executa `bin/console novosga:install` para criar o schema do banco e o usuário admin
4. Inicia todos os serviços

Tempo estimado de primeira inicialização: **2 a 5 minutos** (build das imagens incluso).

> **Warnings normais do Mercure:** nos logs do Mercure aparecem dois avisos sobre HTTP/2 e HTTP/3 exigirem TLS. Eles são **inofensivos** em ambientes HTTP — o Mercure funciona normalmente via HTTP/1.1. Para suprimi-los em produção, configure HTTPS (nginx reverse proxy com certificado).

> **Warnings normais do Mercure:** nos logs do Mercure aparecem dois avisos sobre HTTP/2 e HTTP/3 exigirem TLS. Eles são **inofensivos** em ambientes HTTP — o Mercure funciona normalmente via HTTP/1.1. Para suprimi-los em produção, configure HTTPS (nginx reverse proxy com certificado).

### 1.6 Configuração inicial no painel admin

Acessar `http://[servidor]:8000` com as credenciais definidas em `NOVOSGA_ADMIN_USERNAME` e `NOVOSGA_ADMIN_PASSWORD`.

**Passos obrigatórios antes de usar a triagem:**

1. **Cadastros → Serviços:** cadastrar os tipos de atendimento (ex: "Protocolo", "Certidões", etc.)
2. **Cadastros → Perfis:** criar perfil de atendente
3. **Cadastros → Unidades:** verificar se a unidade foi criada automaticamente
4. **Integrações → Web API:** clicar em **+ Adicionar** para criar um cliente OAuth2
   - Anotar o **Client ID** e **Client Secret** gerados — serão usados na configuração do triage-app (Parte 2)
5. **Dados:** criar um atendente vinculado ao perfil e à unidade

---

## Parte 2 — Triage App (Totem de Triagem)

### 2.1 Origem do projeto

O triage-app é um projeto open source do ecossistema NovoSGA, desenvolvido originalmente em 2018 para rodar como aplicativo desktop (Electron). Para uso em ambiente web/totem via browser, foram necessárias adaptações de compatibilidade — descritas abaixo.

**Repositório original:** http://novosga.org  
**Licença:** MIT

### 2.2 Alterações realizadas

#### Alteração 1 — `src/index.ejs`
**Problema:** O template HTML gerava um `<script>` inline com código Node.js (`process.env.NODE_ENV`, `require('path')`, `__dirname`) que era incluído no HTML final e executado pelo browser, causando o erro:
```
Uncaught ReferenceError: process is not defined
```
O webpack substitui referências a `process.env` dentro do bundle JavaScript via `DefinePlugin`, mas **não processa o HTML**. O script inline chegava intacto ao browser.

**Correção:**
```diff
- <% if (!process.browser) { %>
+ <% if (!process.browser && process.env.BUILD_TARGET !== 'web') { %>
```
A condição original `!process.browser` deveria pular o bloco em browsers, mas em Node.js `process.browser` é `undefined`, tornando `!undefined = true` — o bloco era sempre incluído. A condição adicional usa a variável `BUILD_TARGET=web` já presente no processo de build para excluir corretamente o bloco Electron-only.

---

#### Alteração 2 — `.electron-vue/webpack.web.config.js`
**Problema:** O projeto usava `node-sass` v4 para compilar arquivos SASS. Essa versão depende de binários nativos compilados para cada versão do Node.js e **não suporta Node.js 17+**. Além disso, a API do `sass-loader` v7 usada é incompatível com o `sass-loader` v10 necessário para o Dart Sass.

**Correção 1 — regra de processamento `.sass`:**
```diff
- use: ['vue-style-loader', 'css-loader', 'sass-loader?indentedSyntax']
+ use: ['vue-style-loader', 'css-loader', {
+   loader: 'sass-loader',
+   options: { sassOptions: { indentedSyntax: true } }
+ }]
```

**Correção 2 — opção `loaders` do vue-loader:**
```diff
  options: {
    extractCSS: true,
-   loaders: {
-     sass: 'vue-style-loader!css-loader!sass-loader?indentedSyntax=1',
-     scss: 'vue-style-loader!css-loader!sass-loader',
-     less: 'vue-style-loader!css-loader!less-loader'
-   }
  }
```
A opção `?indentedSyntax` era a forma antiga (sass-loader v7) de passar configuração. Na v10, as opções passam pelo objeto `sassOptions`. A opção `loaders` foi removida para que o vue-loader delegue o processamento às regras do webpack, que usam a nova API.

---

#### Alteração 3 — `package.json`
**Problema:** `node-sass` v4 estava declarado como dependência e seria instalado no Docker, onde também falharia. Além disso, o script `postinstall` rodava o linter automaticamente após `npm install`, o que falha sem um ambiente completamente configurado.

**Correção:**
```diff
- "node-sass": "^4.9.2",
+ "sass": "^1.99.0",               ← Dart Sass (substituto oficial, sem binários nativos)
+ "sass-loader": "^10.5.2",        ← compatível com webpack 4 + Dart Sass

- "postinstall": "npm run lint:fix"
+ "postinstall:disabled": "npm run lint:fix"   ← renomeado para desativar o hook automático
```

> Renomear para `postinstall:disabled` é preferível a deletar: o npm não reconhece o nome como lifecycle hook e não o executa, mas o script permanece acessível via `npm run postinstall:disabled` se necessário.

---

#### Alteração 4 — `package-lock.json` (regenerar localmente)
**Problema:** O `package-lock.json` original foi gerado com `node-sass` e ficou desatualizado após a troca para `sass`. Com o lockfile desatualizado, `npm install` dentro do Docker precisava re-resolver todas as dependências do zero — um processo pesado que falha silenciosamente em ambientes com proxy SSL corporativo.

**Passo obrigatório — executar uma vez na máquina de desenvolvimento (antes do build Docker):**
```bash
cd triage-app-2.1.0
npm install --legacy-peer-deps
```

Isso regenera o `package-lock.json` consistente com o `package.json` atual. O `npm ci` no Docker usará esse lockfile e instalará os pacotes sem re-resolver dependências.

> O `node_modules/` gerado localmente é ignorado pelo Docker via `.dockerignore`.

---

#### Alteração 5 — `Dockerfile`
**Problema:** O Dockerfile original usava `node:8` (suporte encerrado em 2019). Além disso, em ambientes com **proxy corporativo com inspeção SSL** (certificado auto-assinado), o npm no container Alpine falha silenciosamente com `Exit handler never called!` ao tentar baixar pacotes — a conexão HTTPS ao registry é interceptada e o certificado da CA corporativa não está no store do Alpine.

**Correção:**
```diff
- FROM node:8 AS build
+ FROM node:18-alpine AS build

  WORKDIR /app
  COPY package*.json ./

- RUN npm install && \
-     npm run build:web
+ RUN npm config set strict-ssl false && npm ci --legacy-peer-deps
  COPY . .
+ RUN NODE_OPTIONS=--openssl-legacy-provider npm run build:web

  FROM nginx:alpine
  COPY --from=build --chown=nginx:nginx /app/dist/web /usr/share/nginx/html
  EXPOSE 80
  CMD ["nginx", "-g", "daemon off;"]
```

| Flag / Comando | Motivo |
|---|---|
| `node:18-alpine` | Node.js 18 LTS; Alpine reduz tamanho da imagem de build |
| `npm config set strict-ssl false` | Ambiente com proxy SSL corporativo — o container Alpine não tem o certificado da CA interna |
| `npm ci` | Usa o `package-lock.json` diretamente, sem re-resolver dependências — mais rápido e confiável que `npm install` |
| `--legacy-peer-deps` | Pacotes de 2018 têm conflitos de peerDeps no npm moderno |
| `NODE_OPTIONS=--openssl-legacy-provider` | webpack 4 usa algoritmo MD4 removido no OpenSSL 3 (Node.js 17+) |

> **Aviso de segurança:** `strict-ssl false` desabilita a verificação do certificado SSL apenas dentro do container de *build* (não afeta o nginx de produção). Em redes sem proxy corporativo, esta linha pode ser removida.

---

#### Alteração 6 — `src/renderer/pages/Settings.vue`
**Problema:** Após salvar as configurações e confirmar o diálogo de sucesso, o app permanecia na tela de configurações sem navegar para a tela principal. No Electron original, a navegação era controlada pelo processo principal via IPC — inexistente no contexto web.

**Correção:**
```diff
- this.$swal('Success', 'Configuration Ok', 'success')
+ this.$swal('Success', 'Configuration Ok', 'success').then(() => {
+   this.$router.push('/')
+ })
```

---

### 2.3 Build e execução

O Triage App está incluído no `docker-compose.yml` do NovoSGA. O comando `docker compose up -d` (seção 1.5) já sobe o triage-app junto com os demais serviços — não é necessário nenhum comando separado.

### 2.4 Configuração inicial do triage-app

Acessar `http://[servidor]:8080` — o app abre direto na tela de configurações.

Preencher os campos:
| Campo | Valor |
|---|---|
| **Servidor** | `http://[servidor]:8000` |
| **Usuário** | usuário cadastrado no NovoSGA (atendente ou admin) |
| **Senha** | senha do usuário |
| **Client ID** | gerado na etapa 1.6 (Web API do NovoSGA) |
| **Client Secret** | gerado na etapa 1.6 (Web API do NovoSGA) |

Após salvar o servidor, ir para a aba **Serviços**, selecionar a unidade e habilitar os serviços desejados. Salvar novamente — o app navega para a tela de triagem.

---

## Parte 3 — Comandos de Operação

### Iniciar tudo
```bash
cd novosga-docker
docker compose up -d
```

### Parar tudo
```bash
cd novosga-docker
docker compose down
```

### Ver logs
```bash
cd novosga-docker

# Logs do NovoSGA
docker compose logs -f novosga

# Logs do Mercure
docker compose logs -f mercure

# Logs do triage-app
docker compose logs -f triagem
```

### Atualizar o triage-app após mudanças no código
```bash
cd novosga-docker
docker compose build triagem
docker compose up -d triagem
```

---

## Parte 4 — Adaptações para Produção

Os itens abaixo **devem ser alterados** antes da instalação em servidor de produção:

### Senhas e secrets
| Variável | Valor atual (teste) | Ação |
|---|---|---|
| `POSTGRES_PASSWORD` | `novosga@tribunal` | Trocar por senha forte |
| `NOVOSGA_ADMIN_PASSWORD` | `admin@tribunal` | Trocar imediatamente após primeiro acesso |
| `MERCURE_PUBLISHER_JWT_KEY` | `novosga_mercure_secret_tribunal_2026` | Gerar string aleatória ≥ 32 caracteres |
| `MERCURE_SUBSCRIBER_JWT_KEY` | igual acima | Mesmo valor do publisher |
| `MERCURE_JWT_SECRET` | igual acima | Mesmo valor das chaves do Mercure |

Para gerar um secret seguro no Linux:
```bash
openssl rand -hex 32
```

### URLs
Substituir `localhost` pelo IP fixo ou domínio do servidor em:
- `MERCURE_PUBLIC_URL` no `docker-compose.yml`
- Campo **Servidor** nas configurações do triage-app

### Portas sugeridas para produção
| Serviço | Porta sugerida | Acessível por |
|---|---|---|
| NovoSGA (admin + atendimento) | 8000 ou 80 com proxy | Atendentes (rede interna) |
| Triage App (totem) | 8080 ou 80 com proxy | Totens (rede interna) |
| Mercure Hub | 3000 | NovoSGA + browsers (rede interna) |
| PostgreSQL | apenas interno | Nenhum acesso externo |

---

## Parte 5 — Referências

| Item | Referência |
|---|---|
| NovoSGA — repositório Docker | https://github.com/novosga/docker |
| NovoSGA — site oficial | http://novosga.org |
| Triage App — licença MIT | Autor: Rogerio Lino (rogeriolino@gmail.com) |
| Mercure — protocolo tempo real | https://mercure.rocks |
| Dart Sass (substituto do node-sass) | https://sass-lang.com/dart-sass |
| Node.js 18 LTS | https://nodejs.org |
| PostgreSQL 16 | https://www.postgresql.org |
