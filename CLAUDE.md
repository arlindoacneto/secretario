# CLAUDE.md — Projeto "Secretária Online"

> Documento de contexto para o assistente (Claude) e para qualquer dev que pegar o projeto.
> Última atualização: 2026-06-07.

## 1. O que é

Protótipo de **assistente pessoal / secretária digital** feito sob medida para a operação da **Inprint** (empresa de tecnologia especializada em **ECM e Workflow com BPMN 2.0**, com customização em diversas linguagens).

Objetivo central: **capturar rápido** (por voz ou texto) qualquer compromisso, tarefa ou negócio que aparece no dia a dia, classificar automaticamente e organizar em três visões — **Agenda, Fila de Tarefas e Funil de Vendas**. A premissa de design é "captura primeiro, revisa depois": voz e texto **nunca salvam sozinhos**, sempre caem num modal de confirmação antes de gravar.

O domínio dos dados-semente confirma o negócio: ACME (Implantação ECM + GED), Gama Saúde (Portal + assinatura digital), Beta Logística (Workflow BPMN aprovações), Delta Varejo (Automação de contas a pagar).

## 2. Stack e arquitetura

- **Arquivo único:** `secretaria-online.html` (~727 linhas). HTML + CSS + JS vanilla, zero dependências, zero build, zero backend.
- **Persistência:** `localStorage`, chave `secretaria_v2`. Há migração automática a partir da chave antiga `secretaria_v1` (converte campos `gestor`/`regiao` soltos para o modelo atual de `gestores[]` + `gestorId`).
- **Idioma/locale:** pt-BR (datas, moeda BRL, dias/meses em português).
- **UI:** mobile-first, tema escuro (variáveis CSS em `:root`), navegação por abas fixas no rodapé (Agenda / Tarefas / Funil), botão flutuante "+", modais em overlay, toasts.
- **Voz:** Web Speech API (`SpeechRecognition`/`webkitSpeechRecognition`), `lang=pt-BR`. Degrada com aviso se o navegador não suportar.
- `README.md` está praticamente vazio (só o título). Toda a lógica está no HTML.

## 3. Modelo de dados (objeto `S` no localStorage)

```
S = {
  agenda:   [ {id, title, kind, when(ISO), valor?, notes, att[], srcType?, srcId?, fuId?} ],
  tarefas:  [ {id, title, type, prio, status, att[], followups[]} ],
  funil:    [ {id, cliente, projeto, valor, prob, gestorId, fecha(date), att[], followups[]} ],
  gestores: [ {id, nome, regiao} ]
}
```

- **agenda.kind:** `reuniao` | `pagar` | `retorno`.
- **tarefas.type:** `Análise de documento` | `Retorno de ligação` | `Composição de projeto` | `Geral`.
- **tarefas.prio:** `alta` | `media` | `baixa`. **tarefas.status:** `todo` | `doing` | `done`.
- **followups[]:** `{id, evento(date), desc, retorno(date opcional), reminderId}`. Histórico do processo por tarefa/negócio.
- **att[]:** anexos `{name, data(base64 ou null)}`.
- IDs gerados por `uid()` (timestamp base36 + random).

## 4. Funcionalidades por visão

**Captura (header, sempre visível)**
- Campo de texto + botão de voz + botão "Captar". `parseInput()` interpreta a frase e `classify()` decide o destino (agenda/tarefa/funil).
- Parser extrai: valor (`R$`, "mil/k", "milhão/mi"), probabilidade (`%`), região (MG/SP via palavras-chave), data/hora ("amanhã", "hoje", "depois de amanhã", dias da semana, "dia 10", "15h", "às 14:30").
- `cleanTitle()` remove os tokens já interpretados para sobrar só o título.

**Agenda** — calendário mensal (com bolinhas por tipo de evento e painel do dia selecionado) + visão lista agrupada por dia, com destaque para hoje e atrasados (⚠).

**Fila de Tarefas** — Kanban de 3 colunas (A fazer / Em andamento / Concluído), mover entre status, contagem de follow-ups por card.

**Funil de Vendas** — tabela agrupada por gestor e região, com subtotais por gestor e total geral. KPIs no topo: **Total geral, Ponderado (valor × probabilidade), Nº de negócios, Ticket médio**. Filtro por região (chips). Barra de probabilidade colorida (verde ≥60%, âmbar ≥35%, vermelho abaixo).

**Gestores** — CRUD de gestores e regiões; bloqueia exclusão de gestor que ainda tem negócios.

**Follow-ups + lembretes** — ao registrar um follow-up com "data de retorno", o app **cria automaticamente um lembrete na Agenda** (`kind:'retorno'`) ligado à origem via `srcType`/`srcId`/`fuId`. Excluir o follow-up remove o lembrete; excluir a tarefa/negócio faz cascata nos lembretes.

## 5. Convenções para evoluir o código

- Manter **arquivo único** e **vanilla** salvo decisão explícita em contrário.
- Todo dado novo passa por `save()` → `render()`. `render()` chama `renderAgenda/renderTarefas/renderFunil/updateBadges`.
- Sempre escapar texto do usuário com `esc()` ao injetar em HTML.
- Datas internas em ISO; exibição via helpers (`fmtDay`, `fmtTime`, `fdate`, `BRL`).
- Ao mudar o schema, **incrementar a chave** (`secretaria_v3`) e escrever a migração, como já foi feito de v1→v2.

## 6. Riscos e pontos cegos (honesto — ler antes de evoluir)

1. **PERDA DE DADOS é o risco nº 1.** Tudo vive em `localStorage` de **um navegador, num único aparelho**. Limpar cache, trocar de celular, navegação anônima ou cota estourada = **tudo apaga, sem backup, sem aviso**. Para uma ferramenta que guarda funil real com centenas de milhares de R$ e nomes de clientes, isso é frágil demais para uso sério. Mitigação mínima antes de qualquer uso real: **export/import JSON** (1 botão). Solução adequada: backend/sync.
2. **Anexos estouram a cota.** Arquivos viram base64 dentro do `localStorage` (limite ~5–10 MB no total). Poucos anexos quebram o `save()` silenciosamente. Hoje só arquivos <1,5 MB são guardados de fato; o resto guarda só o nome.
3. **Parser de linguagem natural é regex e vai errar.** A confirmação obrigatória protege contra lixo, mas adiciona fricção e a classificação automática acerta menos do que parece. Regiões estão hard-coded só para MG/SP.
4. **É monousuário disfarçado de multiusuário.** Existe "Gestor", mas não há login, permissão nem acesso do próprio gestor ao seu funil. Não escala para equipe nesse formato.
5. **Sem segurança.** Dados de clientes e valores em texto aberto no navegador. Ok para protótipo de validação; inaceitável para produção.
6. **Pergunta estratégica em aberto:** a Inprint **vende** ECM/Workflow/BPMN. Vale decidir cedo se isto é (a) protótipo descartável só para validar o fluxo, (b) produto interno que migra para a plataforma própria da empresa, ou (c) um SaaS. A arquitetura atual só serve para (a).

## 7. Estado atual

Protótipo funcional rodando 100% no cliente, com dados-semente. Nenhum teste automatizado, nenhum backend, nenhum deploy configurado. Código sem alterações desde 2026-06-02 (commit `ec8f368`); este CLAUDE.md está versionado no repositório.

## 8. Decisões pendentes (bloqueiam a evolução)

- [ ] **Export/import JSON** — mitigação mínima contra perda total de dados. Recomendado implementar ANTES de qualquer uso real. Ainda não feito.
- [ ] **Definir o destino do projeto:** (a) protótipo descartável para validar fluxo, (b) produto interno migrando para a plataforma ECM/Workflow da própria Inprint, ou (c) SaaS. Sem essa decisão, qualquer investimento além do export/import corre risco de ser retrabalho.
- [ ] **README.md vazio** — preencher com instruções mínimas de uso (abrir o HTML no navegador, dados ficam locais).
