---
name: relatorio-sistema
description: Gera um raio-x completo do escritório (AD) num relatório único — entregas/monitor por setor, boletos a receber, resultado financeiro (DRE), mensalidades por regime, incoerências e atividade recente. Use quando o usuário disser "relatório do sistema", "raio-x do escritório", "como está o sistema hoje", "panorama geral", "resumo do escritório", "visão geral do AD" ou pedir um relatório geral consolidado.
---

# Relatório do sistema (raio-x do escritório)

Panorama consolidado do AD, **lendo o Supabase por MCP** (a fonte da verdade do site) — **não precisa abrir o
site nem logar**. Só leitura. Compõe as views de relatório e aponta os skills específicos para aprofundar.
Modelo de dados em `docs/03-Arquitetura/Modelo-de-Dados.md`.

## Entrada
- Escopo: geral (default) ou um **setor**. Competências (defaults):
  - **Monitor/entregas:** mês **anterior** (obrigações trabalhadas em arrears).
  - **Boletos/financeiro:** mês **corrente** (e "a receber" = tudo em aberto).
- Deptos: Contábil=1, Fiscal=2, Pessoal=3, Financeiro=4, Legalização/Societário=5 (ou junte em `departments`).

## Passos (consultas de referência)
1. **Clientes / regime** (`companies`):
   ```sql
   select count(*) total, count(*) filter (where regime is not null) com_regime,
          count(*) filter (where regime is null) sem_regime from public.companies;
   ```
2. **Monitor por setor** (concluído = feitos+enviados), mês anterior (`monitor_progresso` + `departments`):
   ```sql
   select mp.department_id, dep.nome, sum(mp.total) total,
          sum(mp.feitos) feitos, sum(mp.enviados) enviados, sum(mp.pendentes) pendentes
   from public.monitor_progresso mp left join public.departments dep on dep.id = mp.department_id
   where mp.competencia = date_trunc('month', current_date) - interval '1 month'
   group by mp.department_id, dep.nome order by mp.department_id;
   ```
   Multas/atrasadas (`deliveries_monthly`): `sum(atrasadas)`, `sum(multas)` no mesmo mês, por `department_id`.
3. **Boletos** (`boletos_monthly` para o mês; `boletos_view` para a receber/devedores):
   ```sql
   select * from public.boletos_monthly order by competencia desc limit 3;
   select count(*) qtd, coalesce(sum(valor),0) a_receber,
          count(*) filter (where vencido) venc_qtd, coalesce(sum(valor) filter (where vencido),0) venc_valor
   from public.boletos_view where status_pagamento = 'em_aberto';
   select razao, company_cnpj, count(*) qtd, sum(valor) valor from public.boletos_view
   where status_pagamento='em_aberto' group by razao, company_cnpj order by valor desc limit 10;
   ```
4. **Resultado / DRE** do mês (`lancamentos_mensal`); se vier vazio, diga "sem lançamentos no mês":
   ```sql
   select tipo, grupo, sum(total) total from public.lancamentos_mensal
   where competencia = date_trunc('month', current_date) group by tipo, grupo;
   ```
   Calcule receitas, despesas, **resultado** (rec−desp), **margem** (res/rec); **fixas** = grupos Folha/
   Pró-labore/Fixas/Impostos, **variáveis** = Operacionais; **folha** = grupo Folha.
5. **Mensalidades por regime** (mediana dos pares) + nº de defasados → aponta `reajuste-mensalidades`:
   ```sql
   with ult as (select distinct on (b.company_cnpj) b.company_cnpj, b.valor from public.boletos b
     where b.valor is not null order by b.company_cnpj, b.competencia desc nulls last, b.id desc)
   select c.regime, count(*) n, percentile_cont(0.5) within group (order by u.valor) mediana
   from ult u join public.companies c on c.cnpj = u.company_cnpj where c.regime is not null
   group by c.regime order by n desc;
   ```
6. **Incoerências** (`financeiro_incoerencias`) + extrato pendente (`extrato_view`) → aponta `organizar-lancamentos`:
   ```sql
   select tipo, count(*) n from public.financeiro_incoerencias group by tipo order by n desc;
   select count(*) pendentes from public.extrato_view where categorizado = false;
   ```
7. **Atividade recente** (`auditoria_geral`):
   ```sql
   select quando, quem, modulo, acao, empresa from public.auditoria_geral order by quando desc limit 15;
   ```

## Saída (relatório, conciso e acionável)
1. **Cabeçalho** — data; nº de clientes; **regime preenchido X/total** (e nº sem regime).
2. **Monitor por setor** (mês anterior) — por setor: **% concluído** (feitos+enviados ÷ total), pendentes,
   atrasadas/multas. Destaque os setores mais atrasados.
3. **Boletos** — **a receber** (R$ e nº), **recebido no mês**, **vencidos** (R$/nº), **% lidos**; top devedores.
4. **Resultado (DRE)** — receitas − despesas = **resultado** e **margem** do mês; **fixas × variáveis**; **folha**.
5. **Mensalidades por regime** — mediana por regime (Simples/Presumido/Real/Doméstica…) e **nº de defasados**;
   "rode `reajuste-mensalidades` para a lista e as sugestões".
6. **Financeiro — pendências** — incoerências por tipo + transações de extrato a categorizar; "rode
   `organizar-lancamentos` para resolver".
7. **Atividade recente** — quem fez o quê (auditoria).
8. **⚠️ Pontos de atenção** — lista curta e acionável (vencidos a cobrar, guias atrasadas por setor,
   incoerências, **47 sem regime**, etc.).

## Princípios
- **Números reais** das views; **só leitura** (não altera nada).
- **Honesto sobre lacunas:** se o Financeiro ainda não tem lançamentos, diga; idem regime incompleto.
- **Conciso e acionável:** o usuário quer saber **o que está bem e o que fazer**. Para aprofundar, aponte os
  skills específicos: `status-boletos`, `reajuste-mensalidades`, `organizar-lancamentos`, `relatorio-cliente`.
- Relaciona: [[Financeiro-Controle]] · [[Monitor-Obrigacoes]] · [[Analise-Agentes-IA]].
