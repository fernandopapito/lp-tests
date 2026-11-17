# Diagnóstico do pipeline — Lunar Pass

Leia o log do job que falhou **do fim para o começo** e copie a primeira linha de erro real (não o "Process completed with exit code 1"). A ordem de suspeita é: **variável → rede → sandbox → teste**.

## Sintoma → causa → correção

| Sintoma no CI | Causa provável | Correção | Quem |
|---|---|---|---|
| `check-env`: "Cadastre BASE_URL…" | variável ou segredo ausente, ou nome diferente | conferir nomes em *Settings → Secrets and variables → Actions* (`BASE_URL` é Variable, `DATABASE_URL` é Secret) | aluno |
| `check-env`: "DATABASE_URL usa a conexão direta" | segredo com `db.<ref>.supabase.co` | trocar pela string do **Session pooler** | aluno |
| `connect ENETUNREACH` ou `ENOTFOUND` no banco | conexão direta (IPv6) ou host digitado errado | Session pooler; conferir a string no Supabase | aluno |
| `password authentication failed` | senha trocada no Supabase e segredo antigo | atualizar o segredo `DATABASE_URL` | aluno |
| `check-env`: `curl: (6)`, `(7)` ou `(22)` | sandbox fora do ar ou `BASE_URL` errada | abrir a URL; republicar com `yarn dist` + `npx vercel deploy --prebuilt --prod` | aluno |
| `curl: (22)` com HTTP 401, ou a tela de login da Vercel no trace | `BASE_URL` aponta para um deploy de **preview**, protegido por login | usar a URL fixa do projeto (deploy com `--prod`) | aluno |
| Testes de login voltam para `localhost` | URL do sandbox fora das *Redirect URLs* do Supabase | *Authentication → URL Configuration → Redirect URLs* | aluno |
| Página mostra "Supabase não configurado" | sandbox publicado sem a configuração do Supabase | refazer `yarn dist` depois do `yarn setup` e publicar de novo | aluno |
| `browserType.launch: Executable doesn't exist` | cache de browser de outra versão do Playwright | a chave do cache usa o `yarn.lock`; conferir se o lock foi commitado | qa-ct |
| `yarn install --frozen-lockfile` falha | `yarn.lock` desatualizado | rodar `yarn install` local e commitar o `yarn.lock` | aluno |
| `validate` falha em `--list` | import quebrado ou erro de sintaxe num spec | corrigir o teste | qa-test-automation |
| Passa no retry (`flaky` no relatório) | teste instável de verdade | **não** aumentar `retries`; abrir o trace da tentativa que falhou | qa-test-automation (`corrigir`) |
| Falha igual local e no CI, tela diferente do esperado | possível defeito do produto | confirmar com o trace e registrar o defeito | QA (triagem) |
| Artefato vazio ou sem `reports/html` | caminho do `reporter` diferente do upload | alinhar o `path` do upload ao `playwright.config.ts` | qa-ct |
| Agendado nunca roda | cron em horário errado (é UTC), workflow fora da `main`, ou repositório público 60 dias sem commit | converter para UTC; conferir a branch; reativar o workflow na aba *Actions* | aluno |
| Job demora mais de 10 min | `workers: 1` no CI e latência do sandbox | esperado; não mexer em `workers` pelo YAML | — |

## Passo a passo

1. **Ler o log** do job vermelho e classificar: infraestrutura (variável, rede, cache), sandbox (fora do ar, preview, Supabase) ou teste/produto.
2. **Baixar o artefato** `playwright-regression-<n>` e abrir o trace do teste que falhou (`npx playwright show-trace` ou <https://trace.playwright.dev>).
3. **Reproduzir local** com as mesmas variáveis (`BASE_URL` do sandbox no `.env`) antes de mudar o YAML.
4. **Uma mudança por vez** no workflow; rodar com **Run workflow** e conferir.

## O que nunca fazer

- Aumentar `retries` ou pôr `continue-on-error` para "estabilizar".
- Imprimir variáveis no log para "ver se o segredo chegou".
- Mudar `timeout`, `workers` ou o config pelo YAML.
- Colar o log com a string de conexão no chat sem mascarar.
