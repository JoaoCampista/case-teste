---
id: PROD-001
kind: product_spec
title: Plataforma de adoção de gatos e cachorros
parent: CONST-001
version: 0.2.0
status: in_review
related_adrs: [ADR-002]
risk_tier: limited
owners: [ceia]
last_updated: 2026-09-05
spec_hash: pending
status_history:
  - { version: 0.0.1, status: draft, date: 2026-09-05, note: "esqueleto; preenchido no Playbook Bootstrap a partir do case" }
  - { version: 0.1.0, status: draft, date: 2026-09-05, note: "case recebido: plataforma de adoção com login simples e cadastro de animais; aguardando G0" }
  - { version: 0.1.0, status: in_review, date: 2026-09-05, note: "G0 aprovado pelo humano; segue para ADR-002 (stack)" }
  - { version: 0.2.0, status: in_review, date: 2026-09-05, note: "§5 arquitetura preenchida após aceite do ADR-002 (stack TypeScript)" }
---

# PROD-001 — Plataforma de adoção de gatos e cachorros

## 1. Visão

Um mural público de gatos e cachorros disponíveis para adoção, onde qualquer pessoa pode
publicar um animal sob sua responsabilidade e qualquer visitante pode encontrá-lo e falar
diretamente com quem o publicou.

A plataforma **conecta e não intermedia**: ela reduz o custo de tornar um animal visível e o
custo de encontrá-lo. A conversa, a triagem e a entrega acontecem entre as pessoas, fora do
sistema. Esse recorte é deliberado — é o que permite entregar valor real na V1 sem construir
mensageria, contratos de adoção ou verificação de identidade.

## 2. Públicos e jobs-to-be-done

| Público | Job-to-be-done |
|---|---|
| **Responsável** (tutor particular, protetor independente ou voluntário de ONG) — cria conta, publica e mantém as fichas dos animais sob sua guarda | "Quando encontro/resgato um animal que preciso encaminhar, quero publicar a ficha dele em minutos e ser contatado por interessados, para não depender de repostagem em rede social." |
| **Adotante** (visitante, não precisa de conta para navegar) | "Quando decido adotar, quero filtrar por espécie, porte e cidade e ver fotos, para encontrar um animal compatível com minha casa e falar com quem cuida dele." |
| **Responsável, na baixa** — quem já publicou e o animal foi adotado | "Quando o animal é adotado, quero marcar a ficha como adotada, para parar de receber contatos e registrar o desfecho." |

Não há papel de administrador/moderador na V1 (ver §8).

## 3. Pilares

1. **Ficha visível sem atrito.** Publicar um animal é a ação central: formulário curto, sem
   campo obrigatório que o responsável não saiba responder de cabeça.
2. **Busca que respeita a decisão real.** Adoção se decide por espécie, porte, idade, cidade e
   foto — a busca oferece exatamente esses eixos, não mais.
3. **Contato direto e explícito.** O contato do responsável é o call-to-action da ficha, exposto
   com consentimento informado no momento da publicação.
4. **Desfecho registrado.** Toda ficha tem um fim declarado (adotado ou removida); o mural não
   acumula animais fantasma.

## 4. Princípios de design

- **Um só tipo de conta.** Quem publica é o responsável pela ficha. Sem verificação, sem selo,
  sem hierarquia de permissões: a única regra de autorização é "só o autor edita a própria ficha".
- **Leitura é pública, escrita exige login.** Navegar e ver fichas nunca pede conta; publicar,
  editar e dar baixa exigem.
- **Dado pessoal é mínimo e intencional.** Guardamos e-mail (identidade) e o contato que o
  responsável escolheu publicar. Nada de endereço, documento ou geolocalização precisa.
- **Estado explícito, não inferido.** A ficha sai do mural porque alguém marcou o desfecho, nunca
  por expiração silenciosa.
- **Nada de fluxo de adoção no produto.** Sem pedido, sem fila, sem aprovação, sem chat. Se um dia
  entrar, entra como feature nova com ADR — não como evolução acidental.

## 5. Arquitetura (visão)

Decidida em **ADR-002** (aceito no gate G2) e fixada na tabela §2 da Constitution: TypeScript de
ponta a ponta sobre Node 22, Next.js 15 (App Router) renderizando no servidor, SQLite com Drizzle
ORM, Vitest e Biome.

O que o produto impõe à arquitetura, e que o ADR-002 atende:

- **Páginas públicas renderizadas no servidor**, porque o mural é o canal de descoberta e precisa
  ser indexável.
- **Três camadas com fronteira executável:** `core/` (domínio em funções puras, sem framework),
  `storage/` (esquema e consultas, banco injetado), `app/` (rotas, páginas e Server Actions —
  camada fina). `core` não importa `app` nem `storage`; o contrato roda no CI.
- **Duas entidades no núcleo:** **Usuário** (identidade e credencial) e **Animal** (ficha, com
  `responsavel_id` e `status`), mais uma tabela de **sessão**.
- **Login próprio** com e-mail e senha (`scrypt`), sessão no banco referenciada por cookie
  `HttpOnly` — sem provedor externo na V1.
- **Sem armazenamento de binários**: fotos entram como URL de imagem externa, o que remove bucket,
  upload e servidor de estáticos do escopo.
- **Testes herméticos** sem rede e sem banco real: as regras de negócio são testadas em `core/` por
  chamada direta, e a persistência contra SQLite em memória.
- **Piloto em processo único** (contêiner ou VM), não serverless — consequência de SQLite em
  arquivo, registrada no ADR-002.

## 6. Métricas norte (instrumentadas vs aspiracionais)

**Instrumentadas** (deriváveis do próprio banco, sem ferramenta externa):

| Métrica | Definição | Por que é a métrica |
|---|---|---|
| Fichas publicadas | `count(animal)` por semana | mede se o lado da oferta usa a plataforma |
| Taxa de desfecho declarado | `count(status = adotado) / count(animal criado no período)` | mede se o mural reflete a realidade |
| Tempo até o desfecho | mediana de `data_adocao − data_publicacao` | única proxy honesta de "a plataforma ajudou" |
| Fichas ativas com contato válido | `count(status = disponivel)` | tamanho do mural útil |

**Aspiracionais** (não instrumentadas na V1 — exigiriam rastrear cliques ou perguntar ao usuário):

- Adoções efetivamente originadas na plataforma (não sabemos se o contato veio daqui).
- Contatos recebidos por ficha.
- Satisfação do adotante e permanência do animal na nova casa.

Registrar essa separação é intencional: a taxa de desfecho declarado é um limite inferior das
adoções reais, e vamos ler os números sabendo disso.

## 7. Roadmap (features F-*)

| Ordem | F-ID | Escopo | Depende de |
|---|---|---|---|
| 0 | **F-SETUP-V1** | Esqueleto do projeto: estrutura de pastas e fronteiras, runner de testes com 1 teste verde, lint/format, comando de dev, `.gitignore`, CI mínimo (testes + `spec-drift`) e a semente do Design System (DS-00 arquitetura, DS-06 testes, DS-07 nomes) | ADR-002 |
| 1 | **F-AUTH-V1** | Login simples: cadastro com e-mail + senha (hash), login, logout, sessão, e a regra "só o autor edita a própria ficha" | F-SETUP-V1 |
| 2 | **F-ANIMAL-V1** | Cadastro da ficha do animal (espécie gato/cachorro, nome, idade, porte, sexo, cidade, descrição, URLs de foto, contato), edição e remoção pelo autor | F-AUTH-V1 |
| 3 | **F-MURAL-V1** | Mural público: listagem paginada, filtros (espécie, porte, cidade) e página de detalhe da ficha com o contato do responsável | F-ANIMAL-V1 |
| 4 | **F-DESFECHO-V1** | Baixa da ficha: o responsável marca `adotado`, a ficha sai do mural e o desfecho fica registrado com data | F-MURAL-V1 |

Cada linha vira uma spec própria com seu gate G1. A ordem é uma dependência real, não uma
preferência: sem autor não há dono da ficha, sem ficha não há mural, sem mural não há baixa.

## 8. Não-objetivos

Fora do escopo do produto (entrar exige ADR e feature nova, não "só mais um campo"):

- **Fluxo de adoção dentro da plataforma** — pedido, fila, aprovação, recusa, reserva.
- **Mensageria interna / chat.** O contato é externo, por decisão de produto.
- **Upload e hospedagem de imagens.** Fotos são URLs externas na V1.
- **Verificação de ONG, selo de confiança, papéis diferenciados ou moderação de conteúdo.**
- **Denúncia, banimento e painel administrativo.**
- **Outras espécies além de gato e cachorro.**
- **Geolocalização, mapa, raio de distância.** Cidade é texto.
- **Notificações (e-mail, push), recuperação de senha por e-mail, login social.**
- **Doações, apadrinhamento, pagamentos, adoção à distância.**
- **App móvel nativo.**

## 9. NEEDS_CLARIFICATION

- nenhum

## 10. Premissas registradas (inferidas do case, não perguntadas)

Cada premissa abaixo foi assumida para não gastar perguntas; qualquer uma pode ser corrigida antes
do gate G1 da feature correspondente.

1. **Público brasileiro, idioma português, cidade como texto livre.**
2. **Login simples = e-mail + senha com hash**, sessão própria, sem OAuth, sem verificação de
   e-mail e sem recuperação de senha na V1 (perder a senha custa uma conta nova).
3. **Contato publicado é escolha do responsável** (e-mail ou telefone/WhatsApp, campo livre) e é
   visível a qualquer visitante — o formulário diz isso explicitamente antes de publicar.
4. **Navegação pública, sem login**, para que o mural seja encontrável e compartilhável.
5. **Nenhuma moderação prévia**: a ficha aparece no mural assim que publicada.
6. **Estados da ficha:** `disponivel` e `adotado`, mais remoção pelo autor. Sem `reservado`
   (não há fluxo de adoção).
7. **Uma ficha = um animal.** Sem ninhada/lote.
8. **Escala de piloto** (centenas de fichas, não milhões): sem CDN, cache ou busca full-text.
