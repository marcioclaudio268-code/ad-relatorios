---
name: organizar-lancamentos
description: Lê o extrato bancário importado e os lançamentos do escritório, sugere a categorização das transações pendentes no plano de contas e aponta incoerências (duplicidade, lançamento sem competência, transação parada, crédito de honorário não conciliado). Use quando o usuário disser "organiza os lançamentos", "categoriza o extrato", "o que falta lançar", "tem algo estranho no financeiro?", ou logo após importar um OFX.
---

# Organizar lançamentos (financeiro)

Agente do controle financeiro do escritório (**admin**). Depende do módulo [[Financeiro-Controle]].
**Propõe** categorização e correções a partir do que já está gravado; **nunca escreve sem aprovação**.
Modelo de dados em `docs/03-Arquitetura/Modelo-de-Dados.md`.

## Fontes (somente leitura)
- `extrato_view` — transações do OFX (`categorizado`, `conciliado`, `tipo` credito/debito, `descricao`).
- `plano_contas` (ativas) — contas-alvo (`grupo`/`tipo`).
- `lancamentos_view` — lançamentos existentes (base para duplicidade e padrão de valor por conta).
- `financeiro_incoerencias` — **view determinística** com as incoerências já prontas (0018).
- `boletos_view` (em aberto) — para casar crédito ↔ honorário.

## Entrada
- Escopo: tudo ou um banco/conta; período (default: transações ainda **não** categorizadas).

## Passos
1. **Pendentes:** liste `extrato_view` com `categorizado = false` (e, se pedido, o período).
2. **Sugira a conta** de cada transação:
   - Cruze palavras do `descricao`/contraparte com a **tabela de pistas** abaixo → conta do plano.
   - **Débito** → conta de despesa; **crédito** → receita ou conciliação com boleto.
   - Atribua **confiança** (alta / média / baixa) e a **razão** (o que casou).
3. **Concilie créditos** que batem com `boletos_view` em aberto (mesmo valor; nome ~ razão social) →
   sugira `extrato_conciliar` em vez de criar receita nova.
4. **Incoerências:** leia `financeiro_incoerencias` e complemente com análise:
   - **duplicidade** — mesmo valor + data + conta (possível lançamento em dobro).
   - **sem_competencia** — lançamento sem competência (impacta a DRE/mensal).
   - **transacao_pendente** — transação não categorizada há > 30 dias.
   - **credito_nao_conciliado** — crédito com valor de boleto em aberto, ainda solto.
   - **valor fora do padrão** — transação cujo valor destoa do histórico da conta (compare com a
     mediana dos lançamentos do mesmo grupo; sinalize se > 2×).
5. **Resuma** quanto (R$ e nº) já dá para categorizar com **alta confiança** num lote.

## Pistas de categorização (texto da transação → conta)
- `ALUGUEL`, `IMOBILIARIA` → **Aluguel**
- `ENERGIA`, `CEMIG`, `LIGHT`, `ENEL`, `SANEAMENTO`, `AGUA` → **Energia / Água**
- `VIVO`, `CLARO`, `TIM`, `OI`, `NET`, `INTERNET`, `TELEFON` → **Internet / Telefone**
- `DOMINIO`, `ACESSORIAS`, `SOFTWARE`, `SISTEMA`, `ASSINATURA` → **Software / Sistemas**
- `TARIFA`, `CESTA`, `PACOTE`, `IOF`, `MANUT CONTA`, `TED`, `DOC` → **Tarifas bancárias**
- `SALARIO`, `FOLHA`, `PGTO FUNC`, `ORDENADO` → **Folha de pagamento (salários)**
- `FGTS`, `INSS`, `GPS`, `DARF INSS` → **Encargos (FGTS/INSS)**
- `PRO LABORE`, `PROLABORE` → **Pró-labore**
- `VT`, `VR`, `VALE`, `BENEFICIO`, `PLANO SAUDE`, `UNIMED` → **Benefícios (VT/VR/plano)**
- `DAS`, `SIMPLES`, `ISS`, `DARF` → **Impostos do escritório (DAS/ISS)**
- `POSTO`, `COMBUSTIVEL`, `UBER`, `99`, `PEDAGIO` → **Combustível / Transporte**
- `PAPELARIA`, `MATERIAL`, `SUPRIMENTO` → **Material de escritório**
- `GOOGLE ADS`, `META`, `MARKETING`, `IMPULSIONA` → **Marketing / Publicidade**
- `CARTORIO`, `JUNTA`, `TAXA`, `EMOLUMENTO` → **Despesas com clientes (cartório/taxas)**
- crédito que **não** casa boleto → **Outras receitas** (marcar para revisão)
Ajuste as pistas ao plano real (o admin edita o plano de contas). Na dúvida, confiança **baixa**.

## Aplicar (só com aprovação explícita)
Depois do "ok" do usuário, escreva via Edge Function **`financeiro-lancamentos`** (admin, auditada):
- `extrato_lancar` — transação → lançamento na conta sugerida (aceita lote).
- `extrato_conciliar` — crédito ↔ boleto em aberto (dá baixa no boleto).
Nunca aplique sem aprovação; toda escrita fica na `auditoria`.

## Saída (concisa e acionável)
- **Para categorizar (alta confiança):** tabela `transação → conta sugerida` pronta para 1 clique/lote.
- **Revisar:** transações ambíguas / baixa confiança.
- **Incoerências:** lista por tipo, com o que fazer em cada uma.
- **Resumo:** R$ e nº a categorizar, nº conciliável, nº de incoerências.
Ofereça **salvar o relatório** em `docs/` (ex.: `docs/07-Diario/`) ou **aplicar o lote** aprovado.

## Princípios
- **Sugere, não decide:** o usuário aprova antes de qualquer escrita.
- **Determinístico primeiro:** comece pela view `financeiro_incoerencias` e pelas pistas; só então use julgamento.
- **Conciso e verdadeiro:** confiança honesta; o que for incerto vai para "revisar".
- Relaciona: [[Analise-Agentes-IA]] · [[Financeiro-Controle]] · `status-boletos` (cobrança) · `atualizar-ssd`.
