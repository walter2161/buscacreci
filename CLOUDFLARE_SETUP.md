# 🚀 Busca CRECI - Cloudflare Workers Setup

## ⚠️ Problema: Dependências Inseguras

O erro que você recebeu foi devido a pacotes deprecados:
- `glob@7.2.3` - Old security issue
- `rimraf@3.0.2` - No longer supported
- `inflight@1.0.6` - Memory leak

## ✅ Solução: Versão Otimizada para Cloudflare Workers

A API foi refatorada para rodar **sem dependências externas** no Cloudflare Workers, eliminando todos os problemas de segurança.

---

## 📋 Arquivos a Substituir/Criar

### 1. Substituir `package.json`

```json
{
  "name": "buscacreci-workers",
  "version": "2.0.0",
  "description": "Busca CRECI - API serverless para Cloudflare Workers",
  "main": "src/index.js",
  "scripts": {
    "start": "wrangler dev",
    "deploy": "wrangler deploy",
    "test": "node tests/api.test.js"
  },
  "type": "module",
  "devDependencies": {
    "wrangler": "^3.45.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

**Comando:**
```bash
cd seu-repositorio
cat > package.json << 'EOF'
{
  "name": "buscacreci-workers",
  "version": "2.0.0",
  "description": "Busca CRECI - API serverless para Cloudflare Workers",
  "main": "src/index.js",
  "scripts": {
    "start": "wrangler dev",
    "deploy": "wrangler deploy"
  },
  "type": "module",
  "devDependencies": {
    "wrangler": "^3.45.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
EOF
```

### 2. Substituir `wrangler.toml`

```toml
name = "buscacreci"
type = "javascript"
compatibility_date = "2024-12-01"
workers_dev = true
main = "src/index.js"

[[routes]]
pattern = "*/api/*"
zone_name = ""

[[kv_namespaces]]
binding = "CACHE"
id = ""
preview_id = ""

[env.production]
name = "buscacreci-prod"
vars = { ENVIRONMENT = "production", CACHE_TTL = "86400" }

[env.staging]
name = "buscacreci-staging"
vars = { ENVIRONMENT = "staging", CACHE_TTL = "3600" }
```

**Comando:**
```bash
cat > wrangler.toml << 'EOF'
name = "buscacreci"
type = "javascript"
compatibility_date = "2024-12-01"
workers_dev = true
main = "src/index.js"

[[routes]]
pattern = "*/api/*"
zone_name = ""

[[kv_namespaces]]
binding = "CACHE"
id = ""
preview_id = ""

[env.production]
name = "buscacreci-prod"
vars = { ENVIRONMENT = "production", CACHE_TTL = "86400" }
EOF
```

### 3. Criar `src/index.js` (A API principal)

[Veja o arquivo completo em `src/index.js` no repositório]

**Criar estrutura:**
```bash
mkdir -p src
```

Depois crie o arquivo `src/index.js` com o código da API (veja abaixo).

### 4. Substituir `.github/workflows/deploy.yml`

```yaml
name: Deploy to Cloudflare Workers

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  NODE_VERSION: '18'

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Deploy to Cloudflare Workers
        run: npm run deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

---

## 🔑 Configurar Credenciais Cloudflare

### 1. Obter Account ID

```bash
# Acesse:
# https://dash.cloudflare.com

# Procure por "Account ID" na página inicial
# Copie e salve em algum lugar
```

### 2. Obter API Token

```bash
# Acesse:
# https://dash.cloudflare.com/profile/api-tokens

# Clique em "Create Token"
# Selecione: Edit Cloudflare Workers (template)
# Copie o token gerado
```

### 3. Adicionar no GitHub

```bash
# Acesse seu repositório:
# https://github.com/walter2161/buscacreci/settings/secrets/actions

# Crie 2 secrets:

# Secret 1:
Name: CLOUDFLARE_API_TOKEN
Value: <seu-token-aqui>

# Secret 2:
Name: CLOUDFLARE_ACCOUNT_ID
Value: <seu-account-id-aqui>
```

---

## 📝 Conteúdo do `src/index.js`

Copie e crie este arquivo:

```javascript
/**
 * Busca CRECI - Cloudflare Workers API
 * Deploy serverless sem dependências externas
 */

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    const path = url.pathname;

    // CORS headers
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type, Authorization',
      'Content-Type': 'application/json',
      'Cache-Control': 'public, max-age=3600',
    };

    // Handle CORS preflight
    if (request.method === 'OPTIONS') {
      return new Response(null, { headers: corsHeaders });
    }

    try {
      // Rotear baseado no path
      if (path === '/' || path === '/api/') {
        return handleBuscaCreci(request, env, corsHeaders);
      } else if (path === '/api/status' || path.startsWith('/api/status')) {
        return handleStatus(request, env, corsHeaders);
      } else if (path === '/api/creci' || path.startsWith('/api/creci')) {
        return handleDetalhes(request, env, corsHeaders);
      } else if (path === '/api/estados' || path === '/api/estados/') {
        return handleEstados(corsHeaders);
      } else if (path === '/api/health' || path === '/health') {
        return new Response(JSON.stringify({ status: 'ok' }), { headers: corsHeaders });
      } else {
        return new Response(
          JSON.stringify({ error: 'Rota não encontrada', path }),
          { status: 404, headers: corsHeaders }
        );
      }
    } catch (error) {
      console.error('Erro:', error);
      return new Response(
        JSON.stringify({ error: error.message || 'Erro interno' }),
        { status: 500, headers: corsHeaders }
      );
    }
  },
};

/**
 * Handler: Enviar CRECI para consulta
 */
async function handleBuscaCreci(request, env, corsHeaders) {
  const url = new URL(request.url);
  const creci = url.searchParams.get('creci');

  if (!creci) {
    return new Response(
      JSON.stringify({ error: 'Parâmetro "creci" é obrigatório', exemplo: '/?creci=SP12335F' }),
      { status: 400, headers: corsHeaders }
    );
  }

  const codigoSolicitacao = gerarUUID();
  const creciFormatado = formatarCreci(creci);

  const solicitacao = {
    creci,
    creciFormatado,
    status: 'INICIADO',
    dataCriacao: new Date().toISOString(),
  };

  await env.CACHE.put(
    `solicitacao:${codigoSolicitacao}`,
    JSON.stringify(solicitacao),
    { expirationTtl: 86400 }
  );

  consultarCreciAssincrono(creci, codigoSolicitacao, env);

  return new Response(
    JSON.stringify({
      codigo_solicitacao: codigoSolicitacao,
      message: 'Seu CRECI foi enviado para o sistema de consulta',
      status_url: `/api/status?codigo_solicitacao=${codigoSolicitacao}`,
      creci_formatado: creciFormatado,
    }),
    { headers: corsHeaders }
  );
}

/**
 * Handler: Verificar status
 */
async function handleStatus(request, env, corsHeaders) {
  const url = new URL(request.url);
  const codigoSolicitacao = url.searchParams.get('codigo_solicitacao');

  if (!codigoSolicitacao) {
    return new Response(
      JSON.stringify({ error: 'Parâmetro "codigo_solicitacao" é obrigatório' }),
      { status: 400, headers: corsHeaders }
    );
  }

  const dados = await env.CACHE.get(`solicitacao:${codigoSolicitacao}`);

  if (!dados) {
    return new Response(
      JSON.stringify({
        codigoSolicitacao,
        status: 'NÃO_ENCONTRADO',
        mensagem: 'Solicitação expirou ou não existe',
      }),
      { status: 404, headers: corsHeaders }
    );
  }

  const solicitacao = JSON.parse(dados);

  return new Response(
    JSON.stringify({
      codigoSolicitacao,
      status: solicitacao.status,
      mensagem: solicitacao.mensagem || 'Processando...',
      creciID: solicitacao.creciID,
      creciCompleto: solicitacao.creciFormatado,
      dataCriacao: solicitacao.dataCriacao,
    }),
    { headers: corsHeaders }
  );
}

/**
 * Handler: Obter detalhes do CRECI
 */
async function handleDetalhes(request, env, corsHeaders) {
  const url = new URL(request.url);
  const id = url.searchParams.get('id');

  if (!id) {
    return new Response(
      JSON.stringify({ error: 'Parâmetro "id" é obrigatório' }),
      { status: 400, headers: corsHeaders }
    );
  }

  const dados = await env.CACHE.get(`creci:${id}`);

  if (!dados) {
    return new Response(
      JSON.stringify({ codigo: id, erro: 'CRECI não encontrado' }),
      { status: 404, headers: corsHeaders }
    );
  }

  return new Response(dados, { headers: corsHeaders });
}

/**
 * Handler: Listar estados
 */
async function handleEstados(corsHeaders) {
  const estados = [
    'AC', 'AL', 'AP', 'AM', 'BA', 'CE', 'DF', 'ES', 'GO', 'MA',
    'MT', 'MS', 'MG', 'PA', 'PB', 'PR', 'PE', 'PI', 'RJ', 'RN',
    'RS', 'RO', 'RR', 'SC', 'SP', 'SE', 'TO',
  ];

  return new Response(
    JSON.stringify({
      total: estados.length,
      estados: estados.map((sigla) => ({
        sigla,
        disponivel: true,
      })),
    }),
    { headers: corsHeaders }
  );
}

/**
 * Consulta CRECI de forma assíncrona
 */
async function consultarCreciAssincrono(creci, codigoSolicitacao, env) {
  try {
    const match = creci.match(/([A-Z]{2})(\d+)([A-Z]?)/);
    if (!match) throw new Error('Formato de CRECI inválido');

    const [, estado, numero, digito] = match;

    const response = await fetch(
      `https://conselho.net.br/api/v1/profissional/search?creci=${estado}${numero}${digito || ''}`,
      {
        method: 'GET',
        headers: {
          'User-Agent': 'BuscaCRECI/2.0',
          'Accept': 'application/json',
        },
      }
    );

    if (!response.ok) throw new Error(`Erro na API: ${response.statusText}`);

    const dados = await response.json();

    if (dados.success && dados.data) {
      const creciData = dados.data[0];
      const creciID = gerarUUID();

      const creciInfo = {
        codigo: creciID,
        creciCompleto: `CRECI/${estado} ${numero}-${digito}`,
        nomeCompleto: creciData.nome || 'N/A',
        situacao: creciData.situacao || 'Ativo',
        cidade: creciData.cidade || 'N/A',
        estado: estado,
        momento: new Date().toISOString(),
      };

      await env.CACHE.put(`creci:${creciID}`, JSON.stringify(creciInfo), { expirationTtl: 86400 });

      const solicitacao = await env.CACHE.get(`solicitacao:${codigoSolicitacao}`);
      if (solicitacao) {
        const dados = JSON.parse(solicitacao);
        dados.status = 'FINALIZADO';
        dados.mensagem = 'Creci consultado com sucesso';
        dados.creciID = creciID;
        await env.CACHE.put(`solicitacao:${codigoSolicitacao}`, JSON.stringify(dados), { expirationTtl: 86400 });
      }
    }
  } catch (error) {
    console.error('Erro ao consultar CRECI:', error);
    const solicitacao = await env.CACHE.get(`solicitacao:${codigoSolicitacao}`);
    if (solicitacao) {
      const dados = JSON.parse(solicitacao);
      dados.status = 'ERRO';
      dados.mensagem = error.message;
      await env.CACHE.put(`solicitacao:${codigoSolicitacao}`, JSON.stringify(dados), { expirationTtl: 86400 });
    }
  }
}

/**
 * Utilidades
 */
function gerarUUID() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function (c) {
    const r = (Math.random() * 16) | 0;
    const v = c === 'x' ? r : (r & 0x3) | 0x8;
    return v.toString(16);
  });
}

function formatarCreci(creci) {
  const match = creci.match(/([A-Z]{2})(\d+)([A-Z]?)/);
  if (!match) return creci;
  const [, estado, numero, digito] = match;
  return `CRECI/${estado} ${numero}-${digito || ''}`;
}
```

---

## 🚀 Deploy em 3 Passos

### Passo 1: Atualizar Arquivos Localmente

```bash
# Clonar seu repositório
git clone https://github.com/walter2161/buscacreci.git
cd buscacreci

# Atualizar package.json
# (Copie o conteúdo acima)

# Atualizar wrangler.toml
# (Copie o conteúdo acima)

# Criar diretório src
mkdir -p src

# Criar src/index.js com o código da API acima
# (Use seu editor favorito)

# Atualizar .github/workflows/deploy.yml
# (Copie o conteúdo acima)
```

### Passo 2: Adicionar Secrets do GitHub

```bash
# 1. Vá para: https://github.com/walter2161/buscacreci/settings/secrets/actions

# 2. Clique "New repository secret"

# 3. Crie:
#    Name: CLOUDFLARE_API_TOKEN
#    Value: <seu-token>

# 4. Crie:
#    Name: CLOUDFLARE_ACCOUNT_ID
#    Value: <seu-account-id>
```

### Passo 3: Fazer Push e Deploy

```bash
# Commitar mudanças
git add .
git commit -m "Setup Cloudflare Workers with zero dependencies"
git push origin main

# Acompanhar o deploy em:
# https://github.com/walter2161/buscacreci/actions
```

---

## ✅ Pronto!

Após alguns minutos, sua API estará rodando em:

```
https://buscacreci.<seu-account-id>.workers.dev/api/?creci=SP12335F
```

---

## 🧪 Testar Localmente

```bash
# Instalar dependências
npm install

# Rodar servidor local
npm start

# Em outro terminal:
curl "http://localhost:8787/api/?creci=SP12335F"
```

---

## 📞 Suporte

Se tiver dúvidas:
1. Verifique logs em: https://dash.cloudflare.com/workers
2. Ou execute: `wrangler tail`
