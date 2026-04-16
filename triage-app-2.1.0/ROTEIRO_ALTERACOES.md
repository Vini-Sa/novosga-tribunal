# Roteiro de Alterações — NovoSGA Triage Touch v2.1.0
**Elaborado em:** 15/04/2026  
**Finalidade:** Documentar as modificações necessárias para executar o aplicativo via Docker em ambiente web (browser), em substituição ao modo Electron (desktop) para o qual foi originalmente desenvolvido.

---

## Contexto

O projeto **NovoSGA Triage Touch** é um aplicativo de triagem hospitalar desenvolvido em 2018 com Vue.js 2 + Electron. O código-fonte foi escrito para rodar como aplicativo desktop (Electron), utilizando APIs exclusivas do Node.js (`process.env`, `require`, `__dirname`) que não existem em browsers.

Para implantação em ambiente de totem/web sem instalação de software nos terminais, é necessário realizar o build web e servir os arquivos estáticos via servidor HTTP (nginx no Docker).

O arquivo original não roda diretamente a partir do repositório porque:
- Usa `node:8` no Dockerfile (versão de 2017, encerrada em 2019)
- Usa `node-sass` v4 (incompatível com Node.js 17+)
- Usa webpack 4 com algoritmo de hash incompatível com OpenSSL 3
- Contém código Node.js renderizado diretamente no HTML do browser

---

## Repositórios e Pacotes Utilizados

### Projeto original
| Item | Detalhe |
|---|---|
| Nome | NovoSGA Triagem Touch |
| Versão | 2.1.0 |
| Site oficial | http://novosga.org |
| Autor | Rogerio Lino (rogeriolino@gmail.com) |
| Licença | MIT |

### Imagens Docker utilizadas
| Imagem | Versão | Finalidade |
|---|---|---|
| `node` | `18-alpine` (LTS) | Build do projeto dentro do Docker |
| `nginx` | `alpine` | Servidor HTTP para os arquivos estáticos |

Ambas as imagens são oficiais do Docker Hub (`hub.docker.com`).

### Pacotes npm substituídos/adicionados
| Pacote | Ação | Motivo |
|---|---|---|
| `node-sass@^4.9.2` | **Removido** | Incompatível com Node.js 17+ (usa binários nativos compilados) |
| `sass@^1.99.0` | **Adicionado** | Substituto oficial do node-sass (Dart Sass, sem binários nativos) |
| `sass-loader@^10.5.2` | **Atualizado** | Versão compatível com webpack 4 + Dart Sass |

Todos os demais pacotes são os originais do projeto, sem alteração de versão.

---

## Alterações Realizadas

### Arquivo 1 — `src/index.ejs`

**Tipo:** Correção de bug  
**Risco:** Baixo — não altera lógica de negócio

**Problema:**  
O template HTML continha um bloco `<script>` inline com código Node.js (`process.env.NODE_ENV`, `require('path')`, `__dirname`) que era renderizado diretamente no HTML e executado pelo browser. Como o browser não conhece o objeto `process` do Node.js, o app quebrava com o erro:
```
Uncaught ReferenceError: process is not defined
```

O webpack substitui referências a `process.env` dentro do bundle JavaScript (via `DefinePlugin`), mas **não processa HTML**. O script inline chegava intacto ao browser.

**Alteração:**
```diff
- <% if (!process.browser) { %>
+ <% if (!process.browser && process.env.BUILD_TARGET !== 'web') { %>
```

**Explicação:**  
A condição `!process.browser` deveria excluir browsers, mas no Node.js `process.browser` é `undefined`, então `!undefined = true` — o bloco era sempre incluído. A condição adicional `process.env.BUILD_TARGET !== 'web'` usa a variável de ambiente já presente no processo de build web, excluindo corretamente o bloco Electron-only.

---

### Arquivo 2 — `.electron-vue/webpack.web.config.js`

**Tipo:** Atualização de configuração de build  
**Risco:** Baixo — afeta apenas o processo de compilação

**Problema:**  
A configuração do webpack usava a API antiga do `sass-loader` (v7), incompatível com `sass-loader@10` que suporta o Dart Sass.

**Alteração 1 — Regra de processamento de arquivos `.sass`:**
```diff
- use: ['vue-style-loader', 'css-loader', 'sass-loader?indentedSyntax']
+ use: ['vue-style-loader', 'css-loader', {
+   loader: 'sass-loader',
+   options: { sassOptions: { indentedSyntax: true } }
+ }]
```

**Alteração 2 — Opção `loaders` do vue-loader:**
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

**Explicação:**  
A opção `?indentedSyntax` era a forma antiga (v7) de passar configuração para o loader. Na v10, as opções passam pelo objeto `sassOptions`. A opção `loaders` do vue-loader foi removida para que o vue-loader delegue o processamento de estilos às regras do webpack, que passam a usar a nova API corretamente.

---

### Arquivo 3 — `package.json`

**Tipo:** Troca de dependência  
**Risco:** Baixo — substitui pacote deprecated pelo substituto oficial

**Alteração:**
```diff
- "node-sass": "^4.9.2",
+ "sass": "^1.99.0",
+ "sass-loader": "^10.5.2",
```

**Explicação:**  
O `node-sass` foi oficialmente descontinuado em 2022 pelo time do Sass. O substituto recomendado é o `sass` (Dart Sass), que não depende de binários nativos compilados e funciona em qualquer versão moderna do Node.js. O `sass-loader@10` é a última versão que suporta webpack 4 (webpack 5 é requerido a partir da v12).

---

### Arquivo 4 — `Dockerfile`

**Tipo:** Atualização de ambiente de build  
**Risco:** Baixo — não altera o código do aplicativo

**Alteração completa:**
```diff
- FROM node:8 AS build
+ FROM node:18-alpine AS build
  
  WORKDIR /app
  
  COPY package*.json ./
  
- RUN npm install && \
-     npm run build:web
+ RUN npm install --legacy-peer-deps --ignore-scripts
+ 
+ COPY . .
+ 
+ RUN NODE_OPTIONS=--openssl-legacy-provider npm run build:web
  
- COPY . .
- 
  FROM nginx:alpine
  
  COPY --from=build --chown=nginx:nginx /app/dist/web /usr/share/nginx/html
  
  EXPOSE 80
  
  CMD ["nginx", "-g", "daemon off;"]
```

**Explicação de cada mudança:**

| Mudança | Motivo |
|---|---|
| `node:8` → `node:18-alpine` | Node.js 8 encerrou suporte em 2019. Node.js 18 é LTS (suporte até 2025). Alpine reduz o tamanho da imagem de build. |
| `COPY package*.json` antes do `COPY . .` | Boa prática: permite que o Docker reutilize o cache do `npm install` quando apenas código-fonte muda, sem alterar dependências. |
| `--legacy-peer-deps` | Pacotes de 2018 declaram dependências peer com restrições de versão rígidas incompatíveis entre si no npm moderno. Esta flag restaura o comportamento do npm v6. |
| `--ignore-scripts` | Impede que o script `postinstall` do `package.json` execute o linter automaticamente, o que falharia no ambiente Docker por falta de configuração. |
| `NODE_OPTIONS=--openssl-legacy-provider` | webpack 4 usa o algoritmo MD4 para hashing interno, que foi removido do OpenSSL 3 (padrão no Node.js 17+). Esta flag reabilita os algoritmos legados apenas para o processo de build. |

---

### Arquivo 5 — `src/renderer/pages/Settings.vue`

**Tipo:** Correção de comportamento de navegação  
**Risco:** Baixo — apenas adiciona navegação após ação já existente

**Problema:**  
Após salvar as configurações e confirmar o diálogo "Configuration Ok", o aplicativo permanecia na tela de configurações. O usuário precisava navegar manualmente para a tela principal clicando no link "Voltar".

No Electron original, a navegação entre telas era controlada pelo processo principal via IPC (comunicação entre processos), inexistente no contexto web.

**Alteração:**
```diff
- this.$swal('Success', 'Configuration Ok', 'success')
+ this.$swal('Success', 'Configuration Ok', 'success').then(() => {
+   this.$router.push('/')
+ })
```

**Explicação:**  
`$swal` retorna uma Promise que resolve quando o usuário clica em OK no diálogo. Ao encadear `.then(() => this.$router.push('/'))`, o app navega automaticamente para a tela principal após a confirmação. Se os serviços não estiverem configurados, a tela principal redireciona de volta para configurações — comportamento intencional do aplicativo.

---

## Fluxo de Uso Após as Alterações

### Build e implantação
```bash
docker build -t triage-app .
docker run -d -p 80:80 --name triage-app triage-app
```

### Configuração inicial (primeira vez)
1. Acessar o endereço do servidor na porta configurada
2. O app abre diretamente na tela de **Configurações**
3. Aba **Server**: preencher URL do NovoSGA, Client ID, Client Secret, usuário e senha → **Salvar** → clicar OK
4. O app tentará ir para a tela principal e voltará para Configurações (correto — serviços ainda não configurados)
5. Aba **Services**: selecionar a unidade → habilitar os serviços desejados → **Salvar** → clicar OK
6. O app navega para a tela principal de triagem

### Uso normal
- Acessar o endereço do servidor — o app carrega direto na tela de triagem
- As configurações ficam salvas no `localStorage` do browser/totem

---

## Resumo Executivo

| Arquivo modificado | Categoria | Toca em lógica de negócio? |
|---|---|---|
| `src/index.ejs` | Correção de bug de compatibilidade browser | Não |
| `.electron-vue/webpack.web.config.js` | Configuração de build | Não |
| `package.json` | Substituição de dependência descontinuada | Não |
| `Dockerfile` | Ambiente de build e implantação | Não |
| `src/renderer/pages/Settings.vue` | Comportamento de navegação | Não |

Nenhuma alteração modifica autenticação, comunicação com a API do NovoSGA, emissão de senhas ou qualquer regra de negócio do sistema. Todas as mudanças são restritas a infraestrutura de build e compatibilidade de ambiente.
