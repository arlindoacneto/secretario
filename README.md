# Secretária Online

Assistente pessoal / secretária digital sob medida para a operação da **Inprint** (ECM e Workflow com BPMN 2.0).

Captura rápida — por voz ou texto — de qualquer compromisso, tarefa ou negócio do dia a dia. O app classifica automaticamente e organiza em quatro visões: **Agenda**, **Fila de Tarefas**, **Funil de Vendas** e **Arquivo**. Premissa de design: *captura primeiro, revisa depois* — voz e texto nunca salvam sozinhos; sempre passam por um modal de confirmação antes de gravar.

## Onde fica o projeto

Pasta única (fonte de verdade): `OneDrive\Trabalho\Claude\Secretario`. Padrão de organização adotado: `OneDrive\Trabalho\Claude\<nome-do-projeto>`.

## Como abrir

É um **arquivo único, sem instalação e sem servidor**. Basta abrir `secretaria-online.html` num navegador moderno (Chrome, Edge ou Safari atualizados).

- Para a captura por **voz**, use Chrome ou Edge (Web Speech API). Se o navegador não suportar, o app avisa e o resto continua funcionando.
- Funciona offline. Mobile-first, tema escuro.

## Onde ficam os dados (leia com atenção)

Todos os dados vivem no **`localStorage` deste navegador, neste aparelho**. Não há nuvem, login ou sincronização. Na prática, isso significa que **limpar o cache, trocar de celular, usar aba anônima ou estourar a cota do navegador apaga tudo, sem aviso.**

Por isso o app tem um botão **💾 (Dados & Backup)** no topo. Use-o:

1. **⬇️ Baixar backup (.json)** — gera um arquivo com *todos* os dados. **Esta é a única cópia que sobrevive à perda do navegador.** Guarde o arquivo num lugar seguro (a própria pasta do projeto no OneDrive já serve). Faça isso com regularidade — o app cutuca quando passa de 7 dias sem backup.
2. **⬆️ Restaurar de arquivo** — recarrega os dados a partir de um `.json` exportado. Substitui o que estiver no app (com confirmação, e guardando um snapshot do estado atual antes).
3. **Snapshots automáticos** — o app mantém os últimos snapshots do seu trabalho *dentro do mesmo navegador*, como rede de segurança contra exclusões acidentais ou importações erradas. ⚠️ Eles **não** sobrevivem à limpeza do navegador (para isso, use o backup em arquivo) e **não** guardam o conteúdo binário dos anexos, só a estrutura.

O arquivo de backup usa um formato versionado (`{app, schema, exportedAt, data}`), pensado para servir de **ponte de migração** quando o projeto evoluir para um backend.

## Visões

- **Agenda** — calendário mensal (com marcadores por tipo) e visão em lista agrupada por dia; destaca o dia de hoje e atrasados.
- **Fila de Tarefas** — Kanban de três colunas (A fazer / Em andamento / Concluído).
- **Funil de Vendas** — tabela por gestor e região, com KPIs (total, ponderado por probabilidade, nº de negócios, ticket médio) e filtro por região.
- **Arquivo** — itens encerrados (arquivados em vez de excluídos), com filtro por origem, busca e tag de desfecho (🏆 Ganho / ❌ Perdido) no funil.

Compromissos, tarefas e negócios aceitam **anexos** e **follow-ups**; um follow-up com data de retorno cria automaticamente um lembrete na Agenda.

## Limitações conhecidas (protótipo)

- **Monousuário, single-device.** Existe o conceito de "Gestor", mas não há login, permissão nem acesso de cada gestor ao próprio funil.
- **Sem segurança.** Dados de clientes e valores ficam em texto aberto no navegador — aceitável para validação, inaceitável para produção.
- **Anexos grandes (>1,5 MB)** guardam só o nome, para não estourar a cota do `localStorage`.
- **O parser de linguagem natural é baseado em regex** e vai errar às vezes; por isso a confirmação é obrigatória. Regiões reconhecidas automaticamente: MG e SP.

## Roadmap

Decisão tomada (2026-06-08): a Secretária Online seguirá como **produto interno da Inprint**, com migração futura para a plataforma ECM/Workflow/BPMN da própria empresa. O backup em arquivo é o primeiro passo nessa direção. Próximos passos naturais: backend com sincronização, autenticação e multiusuário real.

## Stack

HTML + CSS + JavaScript puro (*vanilla*), arquivo único, sem dependências e sem build. Detalhes de arquitetura, modelo de dados e convenções de evolução estão em [`CLAUDE.md`](./CLAUDE.md).
