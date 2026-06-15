---
name: status-boletos
description: Relatório rápido de status de boletos de honorários por empresa ou geral. Use quando o usuário perguntar "quem não pagou", "quais boletos estão em aberto/vencidos", "quanto temos a receber", "status dos boletos do cliente X", "quem ainda não leu/recebeu o boleto", ou pedir o resumo financeiro do mês.
---

# Status de boletos

Depende do módulo [[Financeiro-Boletos]]. Lê das views `boletos_view` e `boletos_monthly`
(ver `docs/03-Arquitetura/Modelo-de-Dados.md`).

## Entrada
- Escopo: **uma empresa** (CNPJ/nome) ou **geral**.
- Período: competência ou intervalo (default: mês corrente).

## Passos
1. Resolva a empresa (se informada) em `companies` por CNPJ/nome.
2. Consulte os boletos do escopo/período.
3. Classifique por **eixo de pagamento** (em aberto / pago / baixado; **vencido** = em aberto e
   `vencimento < hoje`) e por **eixo de entrega** (não enviado / enviado / lido).
4. Calcule totais em **R$**: a receber (em aberto), recebido no mês, vencido.

## Saída
- **KPIs:** R$ em aberto · R$ pago no mês · % lidos · nº vencidos.
- **Listas acionáveis:** vencidos (cobrar), não enviados (enviar), não lidos (reforçar).
- Se geral: ranking de clientes por valor em aberto.

Mantenha conciso e orientado a ação (o usuário quer saber **o que fazer**). Para exportar em
arquivo, combine com a skill `relatorio-cliente`.
