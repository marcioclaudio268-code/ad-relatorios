# AD — Relatórios (plugin do Claude Code)

Plugin **somente leitura** para rodar os relatórios do hub contábil **AD** direto no Claude Code, sem abrir
o site. Ele consulta o banco do AD (Supabase) via um servidor MCP em **modo read-only** travado no projeto —
**não altera nada**.

## O que vem dentro
Skills (rodam digitando `/ad-relatorios:<nome>`):
- **`relatorio-sistema`** — raio-x geral: monitor por setor, boletos a receber, DRE do mês, mensalidades por
  regime, incoerências e pontos de atenção.
- **`status-boletos`** — quem não pagou, vencidos, a receber, % lidos.
- **`reajuste-mensalidades`** — quem está defasado por regime + sugestões.
- **`organizar-lancamentos`** — incoerências do financeiro / o que falta categorizar.

> Modo **read-only**: o plugin só **lê e sugere**. Aplicar baixas/lançamentos continua no site do AD.

---

## Instalação (o desenvolvedor faz uma vez na máquina do chefe)

**Pré-requisitos:** [Claude Code](https://code.claude.com) instalado + **Node.js** (para o `npx`).

### 1) Criar o token de acesso (read-only já é garantido pelo plugin)
No Supabase → ícone do perfil → **Account → Access Tokens → Generate new token** (nome: `AD relatorios`).
Copie o valor (começa com `sbp_...`). Guarde — só aparece uma vez.

### 2) Salvar o token na máquina (persistente)
No **PowerShell**:
```powershell
setx SUPABASE_ACCESS_TOKEN "sbp_cole_o_token_aqui"
```
Feche e reabra o terminal/Claude Code depois do `setx` (ele só vale em sessões novas).

### 3) Instalar o plugin
Copie a pasta **`ad-marketplace`** para a máquina (ex.: `C:\AD\ad-marketplace`). No Claude Code:
```
/plugin marketplace add C:\AD\ad-marketplace
/plugin install ad-relatorios@ad-relatorios-br
```
(ou, se publicar num repositório GitHub: `/plugin marketplace add SEU-USUARIO/ad-relatorios`)

Reabra o Claude Code para carregar o MCP.

---

## Uso (o chefe)
Em qualquer chat do Claude Code, digite:
```
/ad-relatorios:relatorio-sistema
```
ou em linguagem natural: *"me dá um relatório do sistema hoje"*. Outros:
- `/ad-relatorios:status-boletos` — *"quem não pagou?"*
- `/ad-relatorios:reajuste-mensalidades` — *"quem está defasado?"*
- `/ad-relatorios:organizar-lancamentos` — *"tem algo estranho no financeiro?"*

---

## Segurança
- **Somente leitura:** o MCP roda com `--read-only` → só `SELECT`, nenhuma escrita.
- **Travado no projeto AD:** `--project-ref=wccuwsupevsezxlqfqsp` e `--features=database` (só ferramentas de
  consulta).
- **Token fora do plugin:** fica na variável de ambiente `SUPABASE_ACCESS_TOKEN` da máquina — **nunca** é
  commitado nem incluído no pacote. Para revogar o acesso, basta apagar o token no Supabase.

## Atualizar
Substitua a pasta `ad-marketplace` (ou dê `git pull` no repo) e rode `/plugin update ad-relatorios`.
