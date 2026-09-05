---
id: PLAN-F-SETUP-V1
kind: plan
title: Plano — F-SETUP-V1
parent: F-SETUP-V1
version: 0.1.0
status: in_review
last_updated: 2026-09-05
---

# Plano

## Estratégia

TDD invertido pela natureza da feature: aqui o "teste que falha primeiro" quase sempre é um fence de
arquitetura que reprova o repositório enquanto o artefato não existe. A ordem abaixo aproveita isso —
cada passo escreve o fence, vê a suíte vermelha pelo motivo certo, e só então cria o arquivo que a
deixa verde.

Duas ordens são obrigatórias e não são gosto:

1. **O andaime mínimo vem antes de qualquer fence**, porque sem Vitest instalado não existe teste
   que falhe. Passo 1 é a única etapa sem teste anterior; ela é validada por AC-1.
2. **O Design System vem depois do esqueleto**, nunca antes. DS-00/06/07 descrevem o que os passos
   1–9 de fato construíram. Escrevê-los antes seria inventar convenção para código inexistente — o
   erro que o ADR-001 manda evitar.

Risco de processo a vigiar durante a execução: esta feature toca muitos arquivos de configuração, e
configuração convida a "aproveitar e já deixar pronto" coisas de F-AUTH. Nada de entidade de produto
entra aqui; o esquema Drizzle nasce válido e vazio.

## Arquivos tocados

Criados:

- `package.json`, `package-lock.json`, `tsconfig.json`, `next.config.ts`, `biome.json`,
  `vitest.config.ts`, `tests/setup.ts`, `.dependency-cruiser.cjs`, `drizzle.config.ts`, `.gitignore`
- `src/core/hello.ts`, `src/core/hello.test.ts`
- `src/storage/schema.ts`, `src/storage/db.ts`
- `src/app/layout.tsx`, `src/app/page.tsx`, `src/app/page.test.tsx`
- `tests/arch/{layout,boundaries,no-network,no-real-db,deps-ratchet,tsconfig,no-suppression,gitignore,package-scripts,ci-workflow,design-system}.test.ts`
- `tests/fixtures/boundary-violation/offender.ts`
- `tests/helpers/db.ts` (fixture de banco em memória)
- `drizzle/0000_init.sql`
- `.github/workflows/ci.yml`
- `.specs/design-system/DS-00-architecture.md`, `DS-06-tests.md`, `DS-07-naming.md`

Editados:

- `CLAUDE.md` — seção **Comandos** (projeto) e nova seção **Layout do código**
- `.claude/settings.json` — hooks `PostToolUse` (format) e `Stop` (testes), ao final e como proposta
- `.specs/features/F-SETUP-V1/tasks.md` — marcação de progresso

## Passos (ordem; marque ✅ ao concluir)

1. **Andaime mínimo.** `package.json` com dependências fixadas, `tsconfig.json` em `strict`,
   `vitest.config.ts`, `biome.json`, `next.config.ts`, `.gitignore`. `npm install`. Sem AC própria
   além de AC-1, que fecha no passo 2.
2. Teste de AC-1 (falha: `src/core/hello.ts` não existe) → `src/core/hello.ts` → verde.
3. Teste de AC-12 (falha: não há página) → `src/app/layout.tsx` e `src/app/page.tsx` → verde.
4. Teste de AC-2 (falha: `drizzle/` e `tests/arch/` ainda não existem) → cria a estrutura e
   `src/storage/schema.ts` vazio de entidades, `drizzle/0000_init.sql` → verde.
5. Teste de AC-4a (falha: rede não bloqueada) → `tests/setup.ts` bloqueando `fetch` e socket, ligado
   no `vitest.config.ts` → verde.
6. Teste de AC-4b (falha: não há fixture nem regra de conexão) → `tests/helpers/db.ts` com
   `":memory:"` e `src/storage/db.ts` como único chamador do construtor → verde.
7. Testes de AC-3a e AC-3b (falham: não há contrato) → `.dependency-cruiser.cjs` com as regras e
   `tests/fixtures/boundary-violation/offender.ts` → verde. **Passo mais arriscado da feature**: a
   fixture precisa ficar fora do `src/` que o build compila, e o teste chama a API do
   `dependency-cruiser` em vez do binário, para poder apontá-la a dois alvos diferentes.
8. Testes de AC-5, AC-6, AC-7, AC-8, AC-9 (fences sobre configuração; cada um falha antes do ajuste
   correspondente) → ajustes em `package.json`, `tsconfig.json`, `biome.json`, `.gitignore` → verde.
9. Teste de AC-10 (falha: não há workflow) → `.github/workflows/ci.yml` com os cinco passos → verde.
10. Teste de AC-11 (falha: não há Design System) → `/ds-new` três vezes: DS-00 (arquitetura e
    fronteiras, com a regra de fronteira apontando para o contrato do passo 7), DS-06 (testes:
    co-location, `tests/arch`, naming, hermetismo, comando), DS-07 (nomes: arquivos, identificadores,
    commits) → verde. Máximo 60 linhas cada.
11. **Documentação do repositório.** Preencher a seção **Comandos** do `CLAUDE.md` e acrescentar
    **Layout do código**. Propor os hooks `PostToolUse` e `Stop` no `.claude/settings.json`.
12. **AC-13 na mão.** Clone limpo em diretório temporário, `npm ci`, `npm run build`, `npm run dev`,
    abrir a página. Registrar o resultado no postmortem.
13. Revisão adversarial com o subagente `spec-reviewer` sobre o diff completo; corrigir alta e média.
14. Validação final e commits (ver Verificação).

## Diferidos

- **DS-05 (contrato de erro):** entra na primeira feature que expuser contrato de escrita, por
  gatilho do CLAUDE.md. Criar agora seria convenção sem código.
- **Tabelas `usuario`, `animal` e `session`:** cada uma com a feature que a usa.
- **`npm audit` como passo bloqueante no CI:** o ADR-002 registra a árvore de dependências como
  risco; transformar auditoria em bloqueio exige decidir o que fazer com aviso sem correção
  disponível, e isso é decisão própria.
- **Estilo e componentes de interface:** primeira feature de interface, com ADR se trouxer
  dependência.
- **Deploy e contêiner.**

## Verificação

- `npm run lint` · `npm run typecheck` · `npm test` · `npm run boundaries` · `npm run build`
- `scripts/sdd/spec-lint-all.sh` · `scripts/sdd/spec-index.sh` · `scripts/sdd/spec-drift.sh`
- Contagens recalculadas na hora (`ls -d .specs/features/F-* | wc -l`), nunca copiadas.
- `git add` por caminho, nunca `-A`; `.specs/index.json` adicionado explicitamente.
- Commits: um por grupo coeso de ACs (`feat(setup): …`), um por documento de DS (`docs(ds): …`).
