# Auditoria do projeto — `/qa-ct --audit`

Confira os itens na ordem. Para cada um: **como conferir**, **o que propor** e **o que fica com o usuário**. Mostre os diffs com segredos mascarados (`postgresql://***`) e só aplique depois da aprovação.

Os itens **A1, A2, A3 e A5** são pré-requisitos do `/qa-ct --go`.

## A1 · `baseURL` vem de `BASE_URL`

**Como conferir:** `playwright.config.ts` usa `process.env.BASE_URL` no `baseURL` dos projects `mission-control` e `storefront`. Está em falta se o `baseURL` for um endereço fixo (`http://localhost:3000` ou a URL da Vercel).

**Proposta:** uma constante no topo do config, usada nos dois projects, e remover o `TODO` do storefront.

```ts
const baseURL = process.env.BASE_URL ?? 'http://localhost:3000'
```

```ts
use: {
  ...devices['Desktop Chrome'],
  baseURL,
},
```

Sem a variável, a suíte continua rodando no localhost. Se o `baseURL` estava com a URL da Vercel, essa URL vai para o `.env` (A5).

## A2 · Nenhuma URL completa em `goto`

**Como conferir:** procure em `playwright/` por `goto('http` e por `localhost:3000`. Um `page.goto` com endereço completo ignora o `baseURL` e prende o teste a um ambiente.

**Proposta:** trocar pelo caminho relativo, que o Playwright completa com o `baseURL`.

```ts
await this.page.goto(`/booking/${missionId}`)
```

Não mexa em asserções de URL (`toHaveURL`) além do necessário para o caminho relativo, e não mude o que o teste faz.

## A3 · Senha fora do `db.ts`

**Como conferir:** `playwright/support/db.ts` lê `process.env.DATABASE_URL`, e não há nenhuma string `postgresql://` no repositório (procure em todos os arquivos versionados).

**Proposta:**

```ts
const connectionString = process.env.DATABASE_URL
if (!connectionString) throw new Error('Defina DATABASE_URL (veja .env.example).')
```

E usar `connectionString` no `Pool`, no lugar da string fixa. No diff, mostre a linha antiga como `connectionString: 'postgresql://***'`.

**Atenção (fica com o usuário):** a senha que já foi commitada está **queimada**, porque continua no histórico do Git mesmo depois da correção. O usuário precisa:

1. trocar a senha no Supabase (*Database → Settings → Reset database password*);
2. copiar a string do **Session pooler** (*Connect → Session pooler*), já com a senha nova;
3. colar no `.env` (A5) e, depois, no Secret do GitHub (`/qa-ct --go`).

Não reaproveite a string antiga e não peça a nova no chat.

## A4 · `.env` carregado pelo config

**Como conferir:** o topo do `playwright.config.ts` carrega o `.env` quando ele existe.

**Proposta:**

```ts
import { existsSync } from 'node:fs'

if (existsSync('.env')) process.loadEnvFile()
```

O `process.loadEnvFile()` é do próprio Node, sem instalar o `dotenv`.

**Atenção:** ele exige **Node 20.12 ou mais novo**. Rode `node -v` e, se for mais antigo, peça ao usuário para atualizar o Node antes de aplicar. No CI o workflow usa Node 24.

## A5 · `.env` fora do Git e `.env.example` versionado

**Como conferir:** existe `.env.example` só com os nomes, o `.gitignore` tem `.env`, e o `.env` local tem `BASE_URL` e `DATABASE_URL`.

**Proposta:**

`.env.example` (versionado):

```env
BASE_URL=https://<projeto>.vercel.app
DATABASE_URL=postgresql://postgres.<ref>:<senha>@aws-0-<regiao>.pooler.supabase.com:5432/postgres
```

`.env` (local, fora do Git). Crie só se não existir:

```env
BASE_URL=<a URL do sandbox que estava no config, ou http://localhost:3000>
DATABASE_URL=COLE_AQUI_A_STRING_DO_SESSION_POOLER
```

`.gitignore`: acrescentar `.env`.

Se o `.env` já existir, não sobrescreva: só diga quais nomes faltam. Para saber se o `DATABASE_URL` já foi preenchido, confira sem mostrar o valor: se ainda é `COLE_AQUI_A_STRING_DO_SESSION_POOLER`, está pendente; se contém `@db.` e `.supabase.co`, é a conexão direta e precisa virar a do pooler (**Atenção**).

## A6 · Relatórios em pasta conhecida

**Como conferir:** o `reporter` tem `html` em `reports/html` e `junit` em `reports/junit.xml`, que são os caminhos que o pipeline publica.

**Proposta:**

```ts
reporter: [
  ['list'],
  ['html', { outputFolder: 'reports/html', open: 'never' }],
  ['junit', { outputFile: 'reports/junit.xml' }],
],
```

E acrescentar `reports/` no `.gitignore`.

## A7 · Evidência em toda falha

**Como conferir:** `trace` grava as falhas mesmo sem retentativa. Com `on-first-retry`, o trace só existe quando há `retries`, ou seja, nunca fora do CI.

**Proposta:**

```ts
use: {
  trace: 'retain-on-failure',
  screenshot: 'only-on-failure',
  video: 'retain-on-failure',
},
```

Não mexa em `retries`, `workers`, `timeout` nem `forbidOnly`.

## Depois de aplicar

1. `npx playwright test --list` precisa listar os testes sem erro (prova que o config e os imports carregam).
2. Se o `DATABASE_URL` do `.env` já está preenchido, ofereça rodar a suíte contra o sandbox e mostre o resultado real.
3. Próximo passo: **`/qa-ct --go`**.

## Fora do escopo da auditoria

- Teste instável, clique antes da hidratação, seletor frágil: é com a `qa-test-automation`, modo `corrigir`. Se aparecer na execução, registre como observação e encaminhe.
- Publicar ou republicar o sandbox na Vercel.
