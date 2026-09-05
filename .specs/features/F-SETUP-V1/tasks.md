---
id: TASKS-F-SETUP-V1
kind: tasks
title: Tarefas — F-SETUP-V1
parent: F-SETUP-V1
version: 0.1.0
status: in_review
last_updated: 2026-09-05
---

# Tarefas (toda task aponta para 1+ AC; task sem AC = super-decomposição)

| ID | Descrição | AC | Status |
|---|---|---|---|
| T-001 | `package.json` com dependências diretas fixadas (sem `^`), scripts `dev`/`build`/`test`/`lint`/`format`/`typecheck`/`boundaries`; `npm install` | AC-5, AC-9 | todo |
| T-002 | `tsconfig.json` em `strict: true`, sem desligar checagem individual; `next.config.ts` | AC-6 | todo |
| T-003 | `vitest.config.ts` apontando para `tests/setup.ts` e incluindo teste co-localizado e `tests/arch` | AC-1 | todo |
| T-004 | `biome.json` cobrindo lint e formatação | AC-7 | todo |
| T-005 | `.gitignore` com `node_modules`, `.next`, `*.db`, `*.sqlite`, `.env*` | AC-8 | todo |
| T-006 | `src/core/hello.test.ts` (falha) → `src/core/hello.ts` → verde | AC-1 | todo |
| T-007 | `src/app/page.test.tsx` (falha) → `src/app/layout.tsx` + `src/app/page.tsx` renderizando no servidor → verde | AC-12 | todo |
| T-008 | `tests/arch/layout.test.ts` (falha) → estrutura `src/{core,storage,app}`, `tests/arch`, `drizzle`; `src/storage/schema.ts` válido e sem entidades de produto; `drizzle/0000_init.sql` | AC-2 | todo |
| T-009 | `tests/arch/no-network.test.ts` (falha) → `tests/setup.ts` bloqueando `fetch` e socket com erro nomeado | AC-4a | todo |
| T-010 | `tests/arch/no-real-db.test.ts` (falha) → `tests/helpers/db.ts` com `":memory:"` + migrações; `src/storage/db.ts` como único chamador do construtor | AC-4b | todo |
| T-011 | `tests/arch/boundaries.test.ts` (falha) → `.dependency-cruiser.cjs` com as regras do ADR-002, executado pela API sobre `src/` | AC-3a | todo |
| T-012 | `tests/fixtures/boundary-violation/offender.ts` + assertiva de que o contrato reprova a fixture | AC-3b | todo |
| T-013 | `tests/arch/deps-ratchet.test.ts` comparando dependências diretas com lista literal no próprio teste | AC-5 | todo |
| T-014 | `tests/arch/tsconfig.test.ts` verificando `strict` e ausência de escapatória | AC-6 | todo |
| T-015 | `tests/arch/no-suppression.test.ts` varrendo `biome-ignore`, `@ts-ignore`, `@ts-expect-error` sem justificativa | AC-7 | todo |
| T-016 | `tests/arch/gitignore.test.ts` verificando padrões e ausência de arquivo rastreado que case com eles | AC-8 | todo |
| T-017 | `tests/arch/package-scripts.test.ts` verificando os sete comandos e a ferramenta correspondente | AC-9, AC-13 | todo |
| T-018 | `tests/arch/ci-workflow.test.ts` (falha) → `.github/workflows/ci.yml` com os cinco passos, sem `continue-on-error` | AC-10 | todo |
| T-019 | `tests/arch/design-system.test.ts` (falha) → `/ds-new` DS-00 arquitetura e fronteiras (≤60 linhas, enforcement apontando o contrato de T-011) | AC-11 | todo |
| T-020 | `/ds-new` DS-06 testes: co-location, `tests/arch`, naming, hermetismo, comando (≤60 linhas) | AC-11 | todo |
| T-021 | `/ds-new` DS-07 nomes: arquivos, identificadores, commits (≤60 linhas) | AC-11 | todo |
| T-022 | `CLAUDE.md`: preencher seção **Comandos** e acrescentar **Layout do código** | AC-9 | todo |
| T-023 | Propor hooks `PostToolUse` (format) e `Stop` (testes) em `.claude/settings.json` | AC-7, AC-1 | todo |
| T-024 | AC-13 na mão: clone limpo, `npm ci`, `npm run build`, `npm run dev`, abrir a página; registrar no postmortem | AC-13 | todo |
| T-025 | Revisão adversarial com `spec-reviewer` sobre o diff; corrigir alta e média | AC-1..AC-13 | todo |
| T-026 | Validação final (lint, typecheck, testes, boundaries, build, `spec-lint-all`, `spec-index`, `spec-drift`) e commits | AC-1..AC-13 | todo |
