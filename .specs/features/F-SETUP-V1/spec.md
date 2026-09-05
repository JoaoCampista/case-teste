---
id: F-SETUP-V1
kind: feature_spec
title: Esqueleto do projeto em TypeScript com fences executáveis e semente do Design System
parent: PROD-001
version: 0.1.0
status: in_review
risk_tier: limited
owners: [ceia]
deps: []
related_adrs: [ADR-002]
last_updated: 2026-09-05
spec_hash: pending
status_history:
  - { version: 0.1.0, status: draft, date: 2026-09-05, note: "criação; primeira feature do roadmap, materializa o ADR-002" }
  - { version: 0.1.0, status: in_review, date: 2026-09-05, note: "G1 aprovado pelo humano; feature ativa, implementação liberada" }
---

# F-SETUP-V1 — Esqueleto do projeto em TypeScript com fences executáveis e semente do Design System

> O repositório tem governança (Constitution, ADR-001, ADR-002) e nenhum código. Nenhuma feature do
> roadmap pode começar sem um lugar para escrever, um comando que diga "verde" e uma fronteira que
> impeça a regra de negócio de vazar para dentro do framework. Esta feature entrega esse esqueleto e,
> como o ADR-002 é uma decisão sem execução até que exista um teste que a cobre, entrega também os
> fences que o tornam verificável e a semente do Design System que o descreve.

## WHAT/HOW

### WHAT

O repositório passa a ter uma aplicação TypeScript que sobe, uma suíte de testes que roda em
segundos sem rede e sem banco real, um comando único de lint e formatação, um contrato de fronteira
entre camadas que falha quando violado, e três documentos de Design System descrevendo as
convenções que o esqueleto de fato adota.

Ao fim desta feature, quem for implementar F-AUTH-V1 encontra: onde colocar a regra de negócio,
onde colocar o SQL, como nomear o teste, qual comando confirma que está verde, e um contrato
automático que reclama se a regra for escrita no lugar errado.

Nenhum comportamento de produto é entregue aqui: não há cadastro, não há mural, não há animal. A
única página é um "hello" que prova que o servidor renderiza.

### HOW

Layout do código, conforme o ADR-002 e detalhado em DS-00:

```
src/
  core/          domínio: funções puras. Não importa next, react, drizzle nem better-sqlite3.
    hello.ts     função trivial, existe para provar que a suíte roda
  storage/       esquema Drizzle e consultas. Recebe a instância do banco por parâmetro.
    schema.ts    tabelas (vazio de entidades de produto nesta feature)
    db.ts        criação da instância: único lugar autorizado a chamar o construtor
  app/           Next.js App Router: rotas, páginas, Server Actions. Camada fina.
    layout.tsx
    page.tsx     a página "hello"
tests/
  arch/          fences: fronteiras, ratchet, ausência de rede e de banco real, config
  fixtures/
    boundary-violation/   módulo que viola a fronteira de propósito, para provar que o
                          contrato não é vacuoso
drizzle/         migrações SQL geradas por drizzle-kit
.github/workflows/ci.yml
```

Fluxo de dependência permitido (o contrato do `dependency-cruiser` reprova qualquer flecha a mais):

```
app/  ──►  core/
  │          ▲
  └──────►  storage/  ──┘   (storage pode usar tipos de core; core não conhece ninguém)
```

Mecânica dos pontos que não são óbvios:

- **Fence que se auto-verifica.** Todo fence desta feature é testado em dois sentidos: o código real
  passa, e um caso que viola de propósito é reprovado. Um contrato que nunca reprovou nada não é
  fence, é decoração — e o Design System proíbe regra sem execução.
- **Sem rede.** O setup do Vitest substitui `globalThis.fetch` e `net.Socket` por algo que levanta
  erro nomeado. Não é configuração de conveniência: é a Constitution §1 virando execução.
- **Sem banco real.** `tests/` só obtém banco por uma fixture que cria `":memory:"` e aplica as
  migrações. Um fence varre `src/core` e `src/storage` procurando chamada ao construtor de
  `better-sqlite3`; só `src/storage/db.ts` pode fazê-la.
- **Ratchet de dependências.** Um teste compara as dependências diretas do `package.json` com uma
  lista literal escrita dentro do próprio teste. Trazer um pacote novo obriga a editar o teste no
  mesmo commit, o que transforma "adicionei uma lib" em decisão visível no diff.
- **Semente do Design System.** DS-00 (arquitetura e fronteiras), DS-06 (testes) e DS-07 (nomes)
  descrevem o que o esqueleto passou a fazer — nunca convenção para código que ainda não existe. Um
  fence verifica que cada regra numerada tem linha correspondente em `## Enforcement`.

## User Stories

- **US-1.** Como pessoa que vai implementar a próxima feature, quero um comando que diga "verde" em
  segundos, para saber se quebrei algo antes de commitar.
- **US-2.** Como pessoa que vai implementar a próxima feature, quero saber sem perguntar onde mora a
  regra de negócio, onde mora o SQL e onde mora o teste, para não decidir isso de novo a cada arquivo.
- **US-3.** Como dono do produto, quero que a fronteira entre domínio e framework seja verificada por
  máquina, para que a escolha do ADR-002 não apodreça em silêncio conforme o projeto cresce.
- **US-4.** Como revisor, quero que adicionar dependência apareça no diff como decisão, para que um
  ORM ou uma biblioteca de autenticação não entrem sem ADR.
- **US-5.** Como pessoa nova no repositório, quero clonar, instalar e ver uma página renderizada,
  para confirmar que o ambiente está correto antes de escrever qualquer código.

## Acceptance Criteria

- **AC-1.** A suíte roda por um comando único (`npm test`), sem rede e sem serviço externo, e termina verde com pelo menos um teste de domínio passando. @test:src/core/hello.test.ts
- **AC-2.** O layout é o do ADR-002: `src/core`, `src/storage`, `src/app`, `tests/arch` e `drizzle` existem, e `src/core` não contém arquivo de componente nem de rota. @test:tests/arch/layout.test.ts
- **AC-3a.** O contrato de fronteira, executado sobre o código real, não reporta violação: nenhum módulo de `src/core` importa de `src/app` ou `src/storage`, e nem `src/core` nem `src/storage` importam `next` ou `react`. @test:tests/arch/boundaries.test.ts
- **AC-3b.** O mesmo contrato, executado sobre a fixture que viola a fronteira de propósito, reporta ao menos uma violação — provando que a regra não é vacuosa. @test:tests/arch/boundaries.test.ts
- **AC-4a.** Uma chamada de rede dentro de um teste falha com erro nomeado em vez de sair para a internet: `fetch` e abertura de socket estão bloqueados no setup da suíte. @test:tests/arch/no-network.test.ts
- **AC-4b.** A fixture de banco entrega instância em memória com as migrações aplicadas, e nenhum arquivo de `src/core` ou `src/storage` chama o construtor de `better-sqlite3` — a exceção única é `src/storage/db.ts`. @test:tests/arch/no-real-db.test.ts
- **AC-5.** O conjunto de dependências diretas do `package.json` é idêntico à lista fixada no teste de ratchet; adicionar, remover ou renomear dependência sem editar o teste falha a suíte. @test:tests/arch/deps-ratchet.test.ts
- **AC-6.** O TypeScript está em modo `strict` sem escapatória: `tsconfig.json` tem `strict: true`, não desliga checagem individual, e `npm run typecheck` passa sem erro. @test:tests/arch/tsconfig.test.ts
- **AC-7.** Lint e formatação passam por um binário único (Biome, via `npm run lint`) e o repositório não contém supressão sem razão escrita ao lado: nenhum `biome-ignore`, `@ts-ignore` ou `@ts-expect-error` desacompanhado de comentário. @test:tests/arch/no-suppression.test.ts
- **AC-8.** O `.gitignore` cobre `node_modules`, `.next`, arquivos de banco (`*.db`, `*.sqlite`) e `.env*`, e nenhum arquivo rastreado casa com esses padrões. @test:tests/arch/gitignore.test.ts
- **AC-9.** O `package.json` expõe os comandos que o CLAUDE.md e o CI usam — `dev`, `build`, `test`, `lint`, `format`, `typecheck` e `boundaries` — e cada um aponta para ferramenta declarada nas dependências. @test:tests/arch/package-scripts.test.ts
- **AC-10.** O workflow de CI executa, sem supressão de falha, os cinco passos que o ADR-002 exige: lint, typecheck, testes, contrato de fronteira e `scripts/sdd/spec-drift.sh`; nenhum passo usa `|| true` ou `continue-on-error`. @test:tests/arch/ci-workflow.test.ts
- **AC-11.** A semente do Design System existe e é coerente: `DS-00-architecture.md`, `DS-06-tests.md` e `DS-07-naming.md` estão em `.specs/design-system/`, passam o lint de spec, têm no máximo 60 linhas cada, e toda regra numerada tem linha correspondente na tabela de enforcement. @test:tests/arch/design-system.test.ts
- **AC-12.** A aplicação renderiza a página inicial no servidor: o componente de página produz HTML contendo o texto de boas-vindas, sem depender de execução no navegador. @test:src/app/page.test.tsx
- **AC-13.** A partir de um clone limpo, `npm ci` instala, `npm run build` conclui sem erro e `npm run dev` sobe o servidor; a existência e a integridade desses comandos são verificadas na suíte, e a execução real fica registrada no postmortem. @test:tests/arch/package-scripts.test.ts

## Risks

Resumo: o risco dominante é o fence virar teatro — contrato que nunca reprova nada, ou regra de
Design System sem execução. As ACs 3b e 11 existem exatamente para isso. O segundo risco é a camada
`app/` atrair regra de negócio, esvaziando `core/`; o terceiro é o esqueleto crescer além do
necessário e virar produto por acidente. Detalhes e mitigações em `risks.md`.

## Out of scope

- Qualquer comportamento de produto: cadastro de usuário, login, ficha de animal, mural, filtros,
  desfecho. Tudo isso são F-AUTH-V1 em diante.
- Tabelas de `usuario`, `animal` e `session` no esquema Drizzle. O esquema nasce nesta feature como
  arquivo válido e sem entidades de produto; cada entidade entra com a feature que a usa.
- Design de interface, folha de estilo, biblioteca de componentes e identidade visual. A página
  "hello" é HTML sem enfeite.
- Deploy, contêiner, infraestrutura, domínio e observabilidade.
- DS-05 (contrato de erro): nasce na primeira feature que expuser contrato de escrita, conforme o
  gatilho do CLAUDE.md — criar agora seria convenção para código inexistente.
- `mypy`-equivalente adicional, análise de segurança estática e `npm audit` como passo bloqueante.
- Hooks `PostToolUse` e `Stop` no `.claude/settings.json`: serão propostos ao fim da feature, quando
  os comandos existirem, e não são AC.

## NFRs

Herdados da Constitution §3. Overrides: nenhum.

Concretizações desta feature, verificáveis pelas ACs acima:

- **Adequação funcional:** 100% das ACs com teste declarado passando (AC-1 a AC-12; AC-13 tem parte manual).
- **Manutenibilidade:** lint verde por um binário único (AC-7); supressão proibida (AC-7); fronteira
  de camadas executável (AC-3a/3b).
- **Confiabilidade:** suíte hermética, sem rede (AC-4a) e sem banco real (AC-4b), portanto
  determinística e repetível em qualquer máquina.
- **Segurança:** segredos fora do repositório por `.gitignore` verificado (AC-8); `strict` sem
  escapatória (AC-6) para que erro de tipo apareça antes da execução.
- **Desempenho:** a suíte completa termina em menos de 30 segundos numa máquina de desenvolvimento;
  se passar disso, é sinal de que algum teste deixou de ser hermético.

## NEEDS_CLARIFICATION

- nenhum

## Premissas registradas (inferidas, não perguntadas)

1. **Gerenciador de pacotes: `npm`**, com `package-lock.json` commitado. É o que vem com Node e não
   exige instalação extra de quem clonar.
2. **CI no GitHub Actions**, porque o repositório vive em `github.com/JoaoCampista/case-teste`.
3. **Testes unitários co-localizados** (`src/core/hello.test.ts` ao lado do módulo) e fences de
   arquitetura em `tests/arch/`. Decisão de DS-06: teste de unidade perto do código que descreve,
   teste sobre o repositório inteiro num lugar só.
4. **Nada de Docker nem de contêiner** nesta feature. Rodar é `npm install && npm run dev`.
5. **Sem biblioteca de UI, sem Tailwind, sem CSS-in-JS.** A primeira feature de interface decide
   isso, com ADR se for dependência.
6. **`app/` fica em `src/app`**, variante suportada pelo Next, para manter todo o código-fonte sob
   `src/`.
7. **Versões fixadas** (sem `^` nem `~`) no `package.json`, conforme o risco de mudança
   incompatível do App Router registrado no ADR-002.
