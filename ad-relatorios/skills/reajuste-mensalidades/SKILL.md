---
name: reajuste-mensalidades
description: Analisa os honorários (mensalidades) dos clientes e sugere reajustes comparando cada cliente com a mediana dos PARES do mesmo regime tributário. Use quando o usuário disser "quem está defasado", "sugere reajuste de mensalidade", "quem cobra abaixo dos pares", "análise de reajuste de honorários", "revisar valores de honorário", ou pedir candidatos a reajuste.
---

# Reajuste de mensalidades (comparação por pares)

Agente do controle financeiro (Financeiro/admin). **Propõe** reajustes; **não** altera nada — a decisão e a
aplicação (no Domínio/boleto) são do escritório. Critério de consenso: **comparação com pares do mesmo
regime tributário** ([[Analise-Agentes-IA]] · [[Financeiro-Controle]]).

## Fontes (somente leitura)
- `boletos` — honorário por cliente (valor do boleto "HONORÁRIO CONTÁBIL", 1/cliente/competência).
- `companies` — **`regime`** (283/330; vem do Acessórias `?registrationData` + Domínio) e `razao`. Junta por **CNPJ**.
- Apoio: aba **Mensalidades** já mostra média/mediana global; média R$ ~694 / mediana R$ 385 (base histórica).

## Entrada
- Escopo: **todos** ou um **regime** específico.
- Parâmetros (defaults): limiar de defasagem **20%** abaixo da mediana dos pares; teto de aumento por vez
  **+40%**; piso opcional (ex.: R$ 385).

## Passos
1. **Honorário atual por cliente:** o valor do boleto **mais recente** de cada `company_cnpj` (mensalidade;
   ignore avulsos/extras se identificáveis).
2. **Pares:** junte com `companies.regime`. Agrupe por **regime**; use só regimes com **≥ 5** clientes
   (senão, confiança baixa).
3. **Alvo do par:** a **mediana** do honorário dos pares (robusta a outliers; cite também a média).
4. **Defasagem:** `def% = (mediana_par − honorário) / mediana_par`. **Candidato** se `def% ≥ limiar`.
5. **Sugestão:** aproximar da **mediana dos pares**, respeitando **teto** (+40%) e piso. Ex.: paga R$ 300,
   mediana do Simples R$ 450 → sugerir ~R$ 420 (teto +40%), com nota "abaixo da mediana".
6. **Confiança:** alta (par grande + defasagem clara) · média · baixa (poucos pares / sem regime).
7. **Sem regime (47) ou sem pares:** compare com a **mediana global** (referência fraca), confiança baixa.

### SQL de apoio (mediana/contagem por regime)
```sql
with ult as (
  select distinct on (b.company_cnpj) b.company_cnpj, b.valor
  from public.boletos b
  where b.valor is not null
  order by b.company_cnpj, b.competencia desc nulls last, b.id desc)
select c.regime, count(*) n,
  percentile_cont(0.5) within group (order by u.valor) mediana,
  round(avg(u.valor), 2) media
from ult u join public.companies c on c.cnpj = u.company_cnpj
where c.regime is not null
group by c.regime order by n desc;
```
(Para os candidatos, repita o `ult` e compare `u.valor` com a mediana do regime do cliente.)

## Saída (concisa e acionável)
- **Resumo por regime:** mediana, média, nº de clientes.
- **Candidatos a reajuste** (ranqueados pela defasagem): cliente · regime · honorário atual · mediana dos
  pares · defasagem (R$ e %) · **sugestão (novo valor, +%)** · confiança · razão curta.
- **Sem comparação:** clientes sem regime / sem pares suficientes (revisar à parte).
- **Potencial:** soma dos aumentos sugeridos (R$/mês e R$/ano).
- Ofereça **salvar** o relatório em `docs/` ou **refinar** (por regime, ajustar limiar/teto/piso).

## Princípios
- **Sugere, não decide:** nada é alterado; o escritório aprova e aplica (Domínio/boleto).
- **Mediana > média** para o alvo (honorários são assimétricos; poucos clientes grandes puxam a média).
- **Realista:** respeite o teto por reajuste e o contexto (inflação como sanidade); não empurre todos ao topo.
- **Confiança honesta:** poucos pares ou sem regime → confiança baixa e explícita.
- Relaciona: `status-boletos` (cobrança) · `organizar-lancamentos` · [[Financeiro-Controle]].
