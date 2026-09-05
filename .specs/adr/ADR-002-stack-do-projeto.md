---
id: ADR-002
kind: adr
title: Adotar TypeScript de ponta a ponta com Next.js, SQLite via Drizzle, Vitest e Biome
parent: PROD-001
version: 1.1.0
status: accepted
date: 2026-09-05
deciders: [joao, ceia]
related: [ADR-001, PROD-001]
supersedes: []
amends: []
last_updated: 2026-09-05
---

# ADR-002 — Adotar TypeScript de ponta a ponta com Next.js, SQLite via Drizzle, Vitest e Biome

## Contexto

O produto (PROD-001, aprovado no G0) é um mural público de animais para adoção. Nada foi escrito
ainda: esta é a primeira decisão técnica do repositório, e a tabela §2 da Constitution está vazia
justamente para ser preenchida aqui.

Duas forças vêm de decisão explícita do dono do produto, e são restrição, não preferência a ser
otimizada:

0. **TypeScript no servidor e no navegador.** Um idioma só, decidido em 2026-09-05. Isso elimina
   de saída qualquer stack de servidor em outra linguagem, por bem avaliada que fosse.

As demais vêm do produto e da Constitution:

1. **Páginas públicas indexáveis.** O mural é o canal de descoberta (PROD-001 §5). O HTML precisa
   sair pronto do servidor; uma aplicação que monta a página no navegador entrega casca vazia ao
   buscador. Isso descarta a combinação mais comum em TypeScript (SPA consumindo API JSON).
2. **Escala de piloto.** Centenas de fichas (PROD-001 §10.8), leitura muito maior que escrita,
   poucos escritores simultâneos.
3. **Duas entidades e nenhum binário.** Usuário e Animal; fotos são URL externa. Sem upload,
   bucket, fila, cache ou busca full-text.
4. **Testes herméticos são regra dura**, sem rede e sem banco real (Constitution §1, CLAUDE.md).
   Nenhuma suíte pode depender de um serviço de pé.
5. **Validação na borda e erro em formato único** são NFR default (Constitution §3).
6. **Superfície pequena por princípio:** "não envelope SDKs maduras", "função pura antes de classe".

A tensão central desta decisão é entre (0)+(1) e (4). Em TypeScript, renderizar no servidor com
componentes leva ao ecossistema React de frameworks, cujas unidades de execução são difíceis de
testar sem navegador. A decisão abaixo não finge que esse custo não existe: ela o isola.

## Decisão

Adotamos, para todas as camadas:

| Camada | Decisão |
|---|---|
| Linguagem | TypeScript 5.x em modo `strict`, sem `any` implícito |
| Runtime | Node.js 26.x (ver `## Revisões`, 2026-09-05: substitui a escolha original por Node 22 LTS) |
| Framework | Next.js 15 (App Router), renderizando no servidor; formulários por Server Actions; um só projeto para servidor e navegador |
| Persistência | SQLite através de `better-sqlite3`, com Drizzle ORM para esquema e consultas tipadas; migrações em SQL geradas por `drizzle-kit` |
| Senhas | `scrypt` do módulo `node:crypto`, com sal por usuário e parâmetros em constante nomeada |
| Sessão | sessão no banco (tabela `session`) referenciada por cookie `HttpOnly`, `SameSite=Lax`, `Secure` em produção |
| Testes | Vitest, sempre contra SQLite em memória |
| Lint / format | Biome (um binário para as duas coisas) |
| Fronteira de módulos | `dependency-cruiser`, com contrato executável no CI |

### Mecânica

- **A lógica não mora no framework.** Três camadas com fronteira declarada, a ser descrita em
  DS-00 no F-SETUP:
  - `core/` — domínio em funções puras de TypeScript, sem `import` de Next, de React ou de banco.
    Recebe dado já validado e devolve resultado ou erro. **É onde vive a maior parte das ACs**, e
    testá-lo é uma chamada de função, sem andaime nenhum.
  - `storage/` — esquema Drizzle e consultas. Nenhuma função abre conexão: recebe a instância do
    banco por parâmetro.
  - `app/` — rotas, páginas, Server Actions e componentes. Camada fina: valida entrada, chama
    `core`, devolve HTML. Pode importar as outras duas; nenhuma delas pode importar ela.
  - Regra de dependência: `core` não importa `app` nem `storage`.
- **Injeção do banco é a fronteira testável.** Em produção, a instância aponta para o arquivo
  SQLite; no teste, para `":memory:"` com o esquema aplicado. É o que faz "sem banco real" ser
  verificável em vez de aspiracional. Drizzle sobre `better-sqlite3` roda em memória de verdade,
  no mesmo processo — foi o motivo de escolhê-lo em vez de Prisma.
- **Validação na borda com Zod**, nos limites de Server Action e route handler. `core` assume dado
  válido e tipado.
- **Erro em formato único**, com o formato documentado na spec da primeira feature que expuser
  contrato e depois codificado em DS-05.
- **Migrações versionadas em SQL** geradas por `drizzle-kit` a partir do esquema em TypeScript,
  commitadas e aplicadas em ordem. O mesmo esquema alimenta o banco em memória dos testes.

## Consequências

- Positivas:
  - Um idioma e um tipo por dado, do formulário ao SQL: o tipo do esquema Drizzle chega ao
    componente sem tradução manual, e renomear coluna quebra a compilação em vez de quebrar em
    produção.
  - Indexação resolvida pelo padrão do framework: as páginas do mural são renderizadas no servidor
    sem esforço adicional, atendendo o pilar de descoberta.
  - A regra de teste hermético é cumprida onde importa: as ACs de domínio testam `core` por
    chamada direta, e as de persistência rodam contra SQLite em memória no mesmo processo. Nenhum
    teste precisa de rede ou de serviço.
  - Zero infraestrutura para desenvolver, para o CI e para o piloto: SQLite é um arquivo, e backup
    é `cp`.
  - Biome cobre lint e formatação num binário, então o hook `PostToolUse` e o CI chamam uma coisa
    só, sem a dupla ESLint + Prettier e sem conflito entre elas.
  - Server Actions permitem formulário HTML nativo com validação no servidor, sem construir uma API
    JSON que ninguém além da própria página consumiria.
- Negativas / o que se PERDE (registro honesto):
  - **Adotamos o framework mais opinativo do conjunto.** Next.js traz roteamento, cache,
    fronteira servidor/cliente e convenção de pastas já decididos. Parte do que o DS-00 deveria
    escolher vem de fábrica, e é exatamente o risco que ADR-001 nos manda vigiar. Mitigação
    parcial: `core/` e `storage/` são código nosso, sem `import` de Next — o framework fica confinado
    a `app/`.
  - **Testar a camada `app/` é caro e ficará deliberadamente raso.** Server Components e Server
    Actions não são funções que se chamem à vontade num teste de unidade. Consequência aceita: a
    cobertura de rota fica em poucos testes de integração por feature, e nenhuma AC de regra de
    negócio será declarada verde a partir da camada `app/`. Se uma regra só puder ser testada por
    ali, ela está no lugar errado.
  - **Superfície de dependência muito maior que o mínimo.** Next, React, Drizzle e Biome trazem uma
    árvore de pacotes que não cabe em revisão humana. `npm audit` no CI é paliativo, não solução.
  - **Sem garantia de LTS no runtime** (ver `## Revisões`): correções entram pela linha atual do
    Node, e ficar para trás deixa de ser opção segura.
  - **Ficamos expostos à evolução do App Router**, cuja API mudou de forma incompatível mais de uma
    vez. Atualizar versão maior será trabalho de verdade, não `npm update`.
  - **SQLite escreve por um escritor de cada vez.** Serve o piloto com folga e nada mais. Crescer
    exige troca de persistência, que por §2 é ADR-v2 e bump `major` — decisão consciente, não
    ajuste de configuração. Some-se a isso que Next em plataforma serverless não convive com
    SQLite em arquivo local: o piloto roda em processo único (contêiner ou VM), e mudar isso é ADR.
  - **`better-sqlite3` é dependência nativa**, então o CI e cada máquina precisam compilar ou
    achar binário pronto. É um ponto de atrito real na instalação.
  - **Login escrito à mão.** Sem `contrib.auth` nem provedor: sal, comparação em tempo constante,
    fixação e invalidação de sessão são responsabilidade nossa, com mais superfície de erro de
    segurança do que usar uma biblioteca auditada.
  - **`scrypt` no lugar de Argon2id.** Argon2id é a recomendação atual; ficamos com `node:crypto`
    para não somar outra dependência nativa. Perda real de margem, trocável sem mudar o contrato do
    repositório de usuários.
  - **Bundle e tempo de build** de uma aplicação React para entregar o que é, no fundo, HTML com
    formulário. Custo consciente do idioma único.

## Alternativas consideradas

- **Fastify + templates (Eta) + SQLite, com TypeScript só de leve no navegador** — rejeitada por
  restrição do dono do produto. Tecnicamente é o encaixe mais limpo com a Constitution: rotas são
  funções comuns, testáveis por chamada direta com `app.inject()`, sem componente de servidor,
  sem cache mágico e com uma fração das dependências. Perde porque "TypeScript no front" ficaria
  reduzido a alguns arquivos de interação, e não é isso que foi pedido. O que se perde ao
  rejeitar: a suíte hermética mais simples possível e a menor superfície de dependência do
  conjunto. Se o custo de teste da camada `app/` se mostrar maior que o previsto, é para cá que
  o ADR-002-v2 deve olhar.
- **React Router v7 em modo framework (ex-Remix), no lugar de Next.js** — rejeitada por pouco, e é
  a alternativa mais forte dentro da restrição. Atenderia igualmente idioma único e renderização
  no servidor, com uma vantagem concreta para nós: `loader` e `action` são funções `async`
  exportadas, que um teste importa e chama com uma requisição montada à mão, sem navegador. Isso
  removeria quase toda a objeção de testabilidade da camada de rota. Perdeu por razão de contexto,
  não de mérito: este repositório é material de treinamento, e Next.js é o que a maior parte de
  quem for continuar o projeto já conhece, com documentação e respostas mais abundantes. O que se
  perde ao rejeitar: rota testável por chamada direta e um framework menor.
- **Next.js com Prisma + Postgres** — rejeitada. Prisma pressupõe um banco real para migrar e para
  testar, e tem passo de geração de cliente; somar Postgres colide de frente com a regra de teste
  hermético e traz um serviço para operar em cada ambiente, desproporcional a duas entidades e
  centenas de fichas. Drizzle sobre SQLite em memória entrega tipagem equivalente sem nada disso.
  O que se perde ao rejeitar: o ecossistema de migração mais maduro do TypeScript e um banco que
  aguenta concorrência de escrita.
- **SPA (Vite + React) consumindo API JSON em Node** — rejeitada. É a arquitetura TypeScript mais
  comum e a mais fácil de testar por camadas, mas quebra o pilar de descoberta: o buscador
  receberia casca vazia, e o mural existe para ser encontrado. O que se perde ao rejeitar:
  separação limpa entre front e back e interatividade sem recarga.
- **Postgres desde o início, no lugar de SQLite** — rejeitada por ora. Evitaria a migração futura,
  mas ou os testes passariam a exigir banco de pé (proibido), ou manteríamos dois caminhos de
  persistência. Preferimos pagar a migração quando houver dado real de uso na mesa.
- **Deno ou Bun como runtime** — rejeitada. Ambos rodam TypeScript sem passo de compilação e
  seriam atraentes, mas Next.js e `better-sqlite3` têm suporte de primeira classe em Node, e
  gastar o orçamento de risco do bootstrap em runtime não é onde ele rende. O que se perde ao
  rejeitar: execução direta de TypeScript e um `toolchain` mais enxuto.

## Riscos

- **A camada `app/` atrair regra de negócio** por ser onde o framework convida a escrever, tornando
  as ACs difíceis de testar e esvaziando `core/`. É o risco número um desta escolha. Mitigação:
  contrato do `dependency-cruiser` mais revisão adversarial (`spec-reviewer`) checando que cada AC
  aponta para teste em `core/` ou `storage/`, salvo integração declarada.
- **Autenticação escrita à mão.** Erros clássicos de sal, comparação, fixação e invalidação de
  sessão. Mitigação: F-AUTH-V1 terá AC com teste para cada um desses pontos; nenhuma será marcada
  verde por inspeção.
- **Acoplamento a SQLite em arquivo fechar a porta de deploy serverless**, descoberto tarde.
  Mitigação: registrado aqui como consequência, e o piloto roda em processo único por decisão, não
  por acidente.
- **Mudança incompatível do App Router** em versão maior. Mitigação: versões fixadas no
  `package.json`, atualização de versão maior tratada como tarefa com spec, nunca como manutenção
  de rotina.
- **SQLite chegar ao limite de escrita antes do previsto.** Mitigação: `WAL` ligado; a métrica de
  fichas publicadas por semana (PROD-001 §6) é o gatilho — subindo uma ordem de grandeza, abre-se
  o ADR-v2 de persistência.
- **Árvore de dependências grande esconder vulnerabilidade.** Mitigação: `npm audit` no CI e
  ratchet de dependências diretas (abaixo).

## Fences contra regressão

Cada regra abaixo nasce como AC de F-SETUP-V1 e como linha de enforcement no Design System; regra
sem execução não entra.

1. **Fronteira de camadas executável.** `dependency-cruiser` com contrato proibindo `core` de
   importar `app` ou `storage`, e proibindo `core` e `storage` de importar `next` ou `react`.
   Roda no CI. É a regra R1 de DS-00; sem isso a separação seria folclore, e com ela a consequência
   negativa mais grave desta decisão fica contida.
2. **Sem rede na suíte.** Setup do Vitest substitui `fetch` e o `net` por algo que levanta erro
   durante os testes. Teste que tentar falar com a rede falha por esse motivo.
3. **Sem banco real na suíte.** A fixture só entrega instância `":memory:"`; um teste verifica que
   nenhum módulo de `core/` ou `storage/` abre conexão por conta própria — só o módulo de
   composição chama o construtor do banco. Guarda a injeção descrita na Mecânica.
4. **`strict` sem escapatória.** `tsc --noEmit` no CI, com `strict: true`; Biome barra `any`
   explícito e `@ts-ignore` sem razão de domínio, conforme a proibição de supressão da Constitution.
5. **Dependências diretas sob ratchet.** Um teste compara as dependências declaradas no
   `package.json` com uma lista fixada no próprio teste; adicionar pacote exige mudar o teste no
   mesmo commit, o que torna impossível trazer um ORM, um framework de UI ou uma biblioteca de
   autenticação sem decisão registrada.
6. **CI mínimo obrigatório.** Workflow rodando `biome ci`, `tsc --noEmit`, `vitest run`,
   `depcruise` e `scripts/sdd/spec-drift.sh`. Sem os cinco verdes, nada entra.
7. **Supressão proibida.** Sem `|| true`, sem `--passWithNoTests`, sem `biome-ignore` nem
   `@ts-expect-error` sem razão de domínio escrita ao lado.

## Revisões

- 2026-09-05: **runtime passa de Node 22 LTS para Node 26.x**, por decisão do dono do produto. Motivo
  de contexto: a máquina de desenvolvimento não tem Node 22, e das versões disponíveis (26.8 alpha,
  26.4 estável, 24.18 LTS) foi escolhida a 26.4. Consequência registrada honestamente: **abrimos mão
  da garantia de LTS** — Node 26 não tem janela de suporte estendido, então correção de segurança
  depende de acompanhar a linha atual, e atualizar deixa de ser opcional. Além disso, `better-sqlite3`
  pode não ter binário pré-compilado para a ABI do Node 26, caindo em compilação a partir do fonte;
  se isso inviabilizar a instalação, a alternativa é o `node:sqlite` da biblioteca padrão, e essa sim
  seria troca de linha de persistência, exigindo ADR-002-v2. Rejeitadas: instalar Node 22 (a opção
  que preservaria a decisão intacta) e usar o Node 24 LTS já presente.
  **Leitura de §2 aplicada aqui, declarada para poder ser contestada:** tratamos isto como *revisão*
  com bump `minor` do ADR e da Constitution, e não como ADR-002-v2 com bump `major`, porque a troca
  ocorre ainda dentro do bootstrap, antes de existir uma linha de código sob a decisão original — não
  há stack em vigor sendo substituída. Se o dono do produto preferir o tratamento formal de troca,
  basta dizer, e o ADR-002-v2 é escrito.
- 2026-09-05: aceito pelo dono do produto no gate G2. Passa a ser lei para código novo.
- 2026-09-05: rascunho inicial propunha Python 3.12 + FastAPI + SQLite. Substituído antes de
  qualquer aceite, ainda em `in_review`, por decisão do dono do produto de usar TypeScript no
  servidor e no navegador. Não é reversão (nada foi aceito nem implementado): a restrição de
  idioma entrou como força de contexto e refez a análise. O custo de teste da camada de framework,
  que motivara a preferência anterior, não desapareceu — passou a ser pago por arquitetura
  (`core/` sem framework) e está registrado em Consequências.
- 2026-09-05: decisão inicial. Preenche a tabela §2 da Constitution pela primeira vez (bump
  `minor` 1.0.0 → 1.1.0, conforme §2 e §9). Extensões desta stack entram aqui; troca de qualquer
  linha da tabela exige ADR-002-v2 com `supersedes` e bump `major` da Constitution.
