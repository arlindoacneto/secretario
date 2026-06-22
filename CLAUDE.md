# CLAUDE.md — Projeto "Secretária Online"

> Documento de contexto para o assistente (Claude) e para qualquer dev que pegar o projeto.
> Última atualização: 2026-06-08 (consolidação de pasta + saída do Git: ver seções 7 e 8).

## 1. O que é

Protótipo de **assistente pessoal / secretária digital** feito sob medida para a operação da **Inprint** (empresa de tecnologia especializada em **ECM e Workflow com BPMN 2.0**, com customização em diversas linguagens).

Objetivo central: **capturar rápido** (por voz ou texto) qualquer compromisso, tarefa ou negócio que aparece no dia a dia, classificar automaticamente e organizar em três visões — **Agenda, Fila de Tarefas e Funil de Vendas** — mais uma quarta visão de **Arquivo**. A premissa de design é "captura primeiro, revisa depois": voz e texto **nunca salvam sozinhos**, sempre caem num modal de confirmação antes de gravar.

O domínio dos dados-semente confirma o negócio: ACME (Implantação ECM + GED), Gama Saúde (Portal + assinatura digital), Beta Logística (Workflow BPMN aprovações), Delta Varejo (Automação de contas a pagar).

## 2. Stack e arquitetura

- **Arquivo único:** `secretaria-online.html` (~958 linhas). HTML + CSS + JS vanilla, zero dependências, zero build, zero backend.
- **Persistência:** `localStorage`, chave `secretaria_v3`. Migração automática em cadeia: v3 ← v2 (adiciona `arquivo[]`) ← v1 (converte `gestor`/`regiao` soltos para `gestores[]` + `gestorId`).
- **Idioma/locale:** pt-BR (datas, moeda BRL, dias/meses em português).
- **UI:** mobile-first, tema escuro (variáveis CSS em `:root`), navegação por abas fixas no rodapé (Agenda / Tarefas / Funil / Arquivo), botão flutuante "+", modais em overlay, toasts.
- **Voz:** Web Speech API (`SpeechRecognition`/`webkitSpeechRecognition`), `lang=pt-BR`. Degrada com aviso se o navegador não suportar.
- **Local do projeto (fonte única de verdade):** `OneDrive\Trabalho\Claude\Secretario`. **Padrão de organização adotado para todos os projetos:** `OneDrive\Trabalho\Claude\<nome-do-projeto>` (subpastas livres dentro de cada projeto).
- **Controle de versão:** **não há mais repositório Git.** O projeto saiu do Git por decisão explícita (2026-06-08). Histórico/rollback agora dependem do **histórico de versões do próprio OneDrive** e dos **backups `.json`** gerados pelo app. Ver riscos na seção 6.

## 3. Modelo de dados (objeto `S` no localStorage)

```
S = {
  agenda:   [ {id, title, kind, when(ISO), valor?, notes, att[], srcType?, srcId?, fuId?} ],
  tarefas:  [ {id, title, type, prio, status, att[], followups[]} ],
  funil:    [ {id, cliente, projeto, valor, prob, gestorId, fecha(date), att[], followups[]} ],
  gestores: [ {id, nome, regiao} ],
  arquivo:  [ {…item original, _src:'agenda'|'tarefas'|'funil', _ts(ISO), _desfecho?:'ganho'|'perdido'} ]
}
```

- **agenda.kind:** `reuniao` | `pagar` | `retorno`.
- **tarefas.type:** `Análise de documento` | `Retorno de ligação` | `Composição de projeto` | `Geral`.
- **tarefas.prio:** `alta` | `media` | `baixa`. **tarefas.status:** `todo` | `doing` | `done`.
- **followups[]:** `{id, evento(date), desc, retorno(date opcional), reminderId}`. Histórico do processo por tarefa/negócio.
- **att[]:** anexos `{name, data(base64 ou null)}`.
- IDs gerados por `uid()` (timestamp base36 + random).

## 4. Funcionalidades por visão

**Captura (header, sempre visível)** — Campo de texto + botão de voz + botão "Captar". `parseInput()` interpreta a frase e `classify()` decide o destino (agenda/tarefa/funil). O parser extrai valor (`R$`, "mil/k", "milhão/mi"), probabilidade (`%`), região (MG/SP via palavras-chave), data/hora ("amanhã", "hoje", "depois de amanhã", dias da semana, "dia 10", "15h", "às 14:30"). `cleanTitle()` remove os tokens já interpretados.

**Agenda** — calendário mensal (bolinhas por tipo de evento + painel do dia) + visão lista agrupada por dia, com destaque para hoje e atrasados (⚠).

**Fila de Tarefas** — Kanban de 3 colunas (A fazer / Em andamento / Concluído), mover entre status, contagem de follow-ups por card.

**Funil de Vendas** — tabela agrupada por gestor e região, com subtotais e total geral. KPIs: Total geral, Ponderado (valor × probabilidade), Nº de negócios, Ticket médio. Filtro por região (chips). Barra de probabilidade colorida (verde ≥60%, âmbar ≥35%, vermelho abaixo).

**Gestores** — CRUD de gestores e regiões; bloqueia exclusão de gestor que ainda tem negócios.

**Follow-ups + lembretes** — ao registrar um follow-up com "data de retorno", o app cria automaticamente um lembrete na Agenda (`kind:'retorno'`) ligado à origem via `srcType`/`srcId`/`fuId`. Excluir o follow-up remove o lembrete; excluir/arquivar a tarefa/negócio faz cascata nos lembretes.

**Finalização: Arquivar ou Excluir** — todo encerramento passa pelo modal `finoverlay` (`openFinish`): **📦 Arquivar** (vai para `S.arquivo` com `_src` + `_ts`) ou **🗑️ Excluir permanentemente** (`hardDelete`). No Funil o botão é ✔️ e o modal pede o **desfecho (🏆 Ganho / ❌ Perdido)**, gravado em `_desfecho`. No Kanban, cards "Concluído" ganham botão 📦 Arquivar de 1 clique.

**Aba Arquivo (4ª aba)** — lista de arquivados por data, chips de filtro por origem, busca por texto, tag de desfecho. Ações: ↩️ Restaurar (se o gestor não existe mais, cai no primeiro) e 🗑️ excluir permanente.

**Dados & Backup (botão 💾 no header → modal `bkpoverlay`)** — três camadas, cada uma com escopo de proteção diferente:
- **Export `.json`** (`exportData`) — baixa arquivo com envelope versionado `{app:'secretaria-online', schema:'v3', exportedAt, data:S}`. É a **única cópia que sobrevive à perda do navegador** e a **ponte de migração** para o backend futuro. Registra `lastExport` em `localStorage['secretaria_meta']`.
- **Import `.json`** (`importData`) — aceita o envelope OU um objeto `S` bruto; valida estrutura, confirma a substituição mostrando contagem e chama `maybeSnapshot(true)` antes de sobrescrever.
- **Snapshots automáticos** (`maybeSnapshot`/`restoreSnapshot`) — rede de segurança contra erros *dentro* do app. Rotativos (máx. 5, throttle de 3 min = 180000 ms), em `localStorage['secretaria_snapshots']`. Usam `liteClone()`: descartam o base64 dos anexos para não agravar a cota. **Não** sobrevivem à limpeza do navegador. `save()` está em try/catch: em estouro de cota avisa por toast (e loga) em vez de falhar em silêncio. Um nudge na carga (`backupOverdue`) cutuca se faz ≥7 dias (ou nunca) que não se baixa backup; dot vermelho no botão 💾 (`updateBkpDot`) sinaliza o mesmo.
- **Chaves fora do schema `S`** (por isso não exigiram bump para v4): `secretaria_snapshots` e `secretaria_meta`.

## 5. Convenções para evoluir o código

- Manter **arquivo único** e **vanilla** salvo decisão explícita em contrário.
- Todo dado novo passa por `save()` → `render()`. `render()` chama `renderAgenda/renderTarefas/renderFunil/renderArquivo/updateBadges`.
- Sempre escapar texto do usuário com `esc()` ao injetar em HTML.
- Datas internas em ISO; exibição via helpers (`fmtDay`, `fmtTime`, `fdate`, `BRL`).
- Ao mudar o schema, **incrementar a chave** (próxima: `secretaria_v4`) e escrever a migração em cadeia, como já foi feito de v1→v2→v3.
- **Validar antes de salvar a versão final:** extrair o `<script>` e rodar `node --check`; conferir que toda função chamada (inclusive em `onclick=`) está definida. Existe um harness headless de fumaça do módulo de backup (ver seção 7).
- **Trabalhe sempre na pasta única** `OneDrive\Trabalho\Claude\Secretario`. Não criar cópias do projeto em outras pastas — foi exatamente isso que gerou o fork histórico (seções 6 e 7).

## 6. Riscos e pontos cegos (honesto — ler antes de evoluir)

1. **PERDA DE DADOS é o risco nº 1 — mitigado em app, mas dependente do usuário.** Tudo vive em `localStorage` de **um navegador, num único aparelho**. Limpar cache, trocar de aparelho, navegação anônima ou cota estourada = **tudo apaga**. Mitigação: **export/import JSON** + snapshots automáticos + nudge de backup (seção 4). ⚠️ O anteparo só vale **se o usuário efetivamente baixar o `.json`** — snapshots vivem no mesmo `localStorage` frágil. Solução adequada continua sendo **backend/sync** (item nº 1 da seção 8).
2. **Sem Git = sem rede de segurança de versão profissional.** Desde 2026-06-08 o projeto não está mais em repositório Git. Não há mais histórico de commits, branches, diff confiável nem remote de backup. **Mitigação atual:** o **histórico de versões do OneDrive** (clique direito → "Histórico de versões" / restaurar versão anterior) cobre parte do rollback, e o **backup `.json`** cobre os dados. ⚠️ É proteção mais fraca e menos auditável que Git — qualquer sobrescrita acidental do `.html` depende do OneDrive ter guardado a versão certa. Avaliar reintroduzir Git **fora** do OneDrive (ver seção 8) se o projeto crescer.
3. **OneDrive + arquivos do projeto = risco de cópias-fantasma e placeholders.** O OneDrive gera cópias de conflito (`*-DESKTOP-*.md`) quando há edição em dois lugares, e mantém arquivos "cloud-only" que ferramentas locais leem como vazios/placeholder. Foi assim que o projeto virou um fork silencioso. **Regra:** uma pasta só, edição num lugar só, e sempre conferir o conteúdo real (não confiar no tamanho/placeholder) antes de sobrescrever ou apagar.
4. **Anexos estouram a cota — `save()` avisa em vez de falhar mudo.** Arquivos viram base64 dentro do `localStorage` (limite ~5–10 MB no total). Hoje só arquivos <1,5 MB são guardados de fato; o resto guarda só o nome. O `save()` está em try/catch e dispara toast em estouro; snapshots usam clone leve (sem base64) para não piorar. Teto baixo — solução real exige storage externo.
5. **Parser de linguagem natural é regex e vai errar.** A confirmação obrigatória protege contra lixo, mas adiciona fricção. Regiões hard-coded só para MG/SP.
6. **É monousuário disfarçado de multiusuário.** Existe "Gestor", mas não há login, permissão nem acesso do próprio gestor ao seu funil. Não escala para equipe nesse formato.
7. **Sem segurança.** Dados de clientes e valores em texto aberto no navegador. Ok para protótipo; inaceitável para produção.

## 7. Estado atual

Protótipo funcional rodando 100% no cliente, com dados-semente. Nenhum backend, nenhum deploy configurado.

**Histórico recente:**
- **2026-06-07:** finalização com Arquivar/Excluir + aba Arquivo (schema v3).
- **2026-06-08 — RECONCILIAÇÃO (incidente anterior).** Havia duas cópias divergentes (um repositório Git em `GitHub\secretario` e uma cópia em `Claude\Secretario`), CLAUDE.md/README desencontrados, índice do Git corrompido (sintoma clássico de Git dentro de OneDrive) e o módulo de backup documentado como "FEITO" sem existir funcional. O HTML foi restaurado a partir do último estado íntegro, o módulo Dados & Backup foi reimplementado do zero (export/import versionado + snapshots rotativos com `liteClone` + `save()` à prova de cota + nudge + dot) e validado com `node --check` e um **harness headless de fumaça (13 asserts passando)** cobrindo seed, snapshot no load, `liteClone`, envelope de export, `secretaria_meta`, `backupOverdue`, rotação ≤5 e `save()` engolindo estouro de cota.
- **2026-06-08 — CONSOLIDAÇÃO DE PASTA + SAÍDA DO GIT (esta mudança).** Por decisão do Arlindo, o projeto foi consolidado numa **pasta única** sob o novo padrão `OneDrive\Trabalho\Claude\Secretario`, e o **repositório Git foi descontinuado**. Antes de qualquer exclusão, cada arquivo foi reconciliado escolhendo a melhor versão: HTML = working copy mais nova (65 KB / 958 linhas, com módulo de backup e 4 abas — validada: tags fechadas, migrações v1→v2→v3); README = versão completa que estava (ironicamente) numa cópia de conflito do OneDrive; CLAUDE.md = versão mais completa (esta). Removidas: as cópias de conflito `*-DESKTOP-*.md`, os arquivos de teste `__probe.tmp`/`__wtest.txt`/`__wtest2.html` e a pasta antiga `GitHub\secretario` (com o `.git`).

Schema `S` inalterado (segue `secretaria_v3`); backup usa chaves separadas (`secretaria_snapshots`, `secretaria_meta`).

## 8. Decisões (status atualizado 2026-06-08)

- [x] **Export/import JSON** — FEITO **e verificado** (não só documentado). Envelope versionado como ponte de migração; import aceita envelope OU `S` bruto, confirma com contagem e tira snapshot antes de sobrescrever.
- [x] **Destino do projeto DEFINIDO: produto interno da Inprint**, migrando depois para a plataforma ECM/Workflow/BPMN própria. O formato de export foi desenhado como ponte; próximo grande passo é backend + auth + multiusuário (sair do localStorage).
- [x] **README.md** — FEITO e preenchido (não é mais stub).
- [x] **Fonte única / fim do fork** — pasta única `OneDrive\Trabalho\Claude\Secretario`. Cópias antigas e de conflito removidas.
- [x] **Saída do Git** — feito por decisão explícita. Trade-off aceito: perda de histórico/remote em troca de simplicidade e fim do conflito Git×OneDrive.
- [ ] **Backend / sync / auth** — principal item em aberto. Enquanto não existir, o backup em arquivo é o único anteparo real dos dados. Decisões a tomar: onde hospedar, modelo de auth, e como reaproveitar a plataforma ECM/Workflow existente em vez de reconstruir do zero.
- [ ] **(Reavaliar) Reintroduzir Git fora do OneDrive** — se o projeto crescer, considerar versionamento profissional num caminho local que o OneDrive **não** sincronize (ou repo remoto direto), para recuperar histórico/rollback sem o risco de corrupção visto antes.
- [ ] **Robustez do parser e cota de anexos** — melhorias incrementais pendentes (parser regex MG/SP; teto de 1,5 MB por anexo). Não bloqueiam.
