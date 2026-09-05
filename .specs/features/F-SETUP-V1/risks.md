---
id: RISK-F-SETUP-V1
kind: risks
title: Riscos — F-SETUP-V1
parent: F-SETUP-V1
version: 0.1.0
status: in_review
last_updated: 2026-09-05
---

# Riscos (≥3, quantificados; risco fechado ganha lápide, não é apagado)

| ID | Sev | Risco | Mitigação | Status |
|---|---|---|---|---|
| R-1 | ALTO | **Fence vacuoso.** O contrato de fronteira passa porque não há código suficiente para violá-lo, e ninguém percebe que ele nunca reprovou nada. Vale para os fences de rede, banco e supressão. | AC-3b torna a prova obrigatória: o contrato é executado contra fixture que viola de propósito e precisa reprovar. Mesmo padrão aplicado a AC-4a (teste que tenta chamar rede e espera erro). | aberto |
| R-2 | ALTO | **Regra de negócio nascer em `app/`**, esvaziando `core/` e tornando as ACs das próximas features difíceis de testar. É o risco número um do ADR-002 e o esqueleto é a última chance de prevenir por construção. | Contrato do `dependency-cruiser` (AC-3a) mais regra R1 de DS-00 com enforcement; DS-06 exige que AC de regra de negócio aponte para teste em `core/` ou `storage/`. `spec-reviewer` checa isso em toda feature. | aberto |
| R-3 | MÉDIO | **Escopo do esqueleto vazar para produto.** Configuração convida a "já deixar pronto" tabela de usuário, formulário de login ou folha de estilo, e o setup vira meia-feature sem spec. | Out of scope explícito na spec; esquema Drizzle nasce válido e sem entidades; `spec-reviewer` compara o diff com o Out of scope antes do commit final. | aberto |
| R-4 | ALTO | **`better-sqlite3` não compilar** na máquina de quem clonar ou no runner do CI (dependência nativa, registrada como atrito no ADR-002), travando o AC-13 e o CI. **Elevado de MÉDIO para ALTO em 2026-09-05**: o runtime passou a Node 26.x (ADR-002 §Revisões), para cuja ABI o pacote pode não ter binário pré-compilado. | AC-13 exige clone limpo com `npm ci`, então o problema aparece na própria feature e não em F-AUTH. Se travar, a implementação para e reporta, em vez de improvisar: a alternativa é o `node:sqlite` da biblioteca padrão, que é troca de linha da tabela §2 e portanto exige ADR-002-v2, nunca ajuste silencioso. | aberto |
| R-5 | MÉDIO | **Testar Server Component ser mais caro que o previsto** (AC-12), levando a instalar biblioteca de teste de navegador e furar o ratchet de dependências no primeiro dia. | A página "hello" é deliberadamente sem estado nem dado: o teste renderiza o componente e inspeciona a string. Se exigir mais que isso, o caminho é reduzir o AC-12 a HTML estático e registrar a lição no postmortem — não trazer dependência sem ADR. | aberto |
| R-6 | MÉDIO | **Design System virar folclore no nascimento**, descrevendo intenção em vez do que o esqueleto faz, ou trazendo regra sem execução. | AC-11 verifica que toda regra numerada tem linha na tabela de enforcement e que cada doc tem no máximo 60 linhas; a ordem do plano põe o DS depois do esqueleto, nunca antes. | aberto |
| R-9 | MÉDIO | **Runtime sem LTS.** Node 26.x não tem janela de suporte estendido, então ficar para trás deixa de ser opção segura e o runner de CI pode nem oferecer essa versão, divergindo do ambiente local. | Versão do Node declarada em `engines` e no workflow; se o runner não oferecer a 26, isso aparece na primeira execução do CI (AC-10) e vira decisão registrada, não ajuste silencioso. | aberto |
| R-7 | BAIXO | **Versões fixadas envelhecerem** e o `npm ci` de um clone futuro falhar por incompatibilidade de plataforma. | Node 22 LTS declarado em `engines`; atualização de versão maior é tarefa com spec, conforme ADR-002. | aberto |
| R-8 | BAIXO | **CI verde enganoso** por passo que falha em silêncio. | AC-10 proíbe `|| true` e `continue-on-error` no workflow, e o próprio teste inspeciona o YAML. | aberto |
