# 🎶 musicoven

> Saiba quando seus artistas e selos favoritos lançam um álbum, no dia em que ele sai, e guarde tudo num catálogo organizado por semana, mês e ano.

O **musicoven** é uma aplicação pessoal (PWA) que acompanha os lançamentos de álbuns dos artistas e selos que você segue, avisa por push no dia do lançamento (e quando um álbum é anunciado) e mantém um histórico consultável do que saiu em cada período.

Ele **não depende de uma assinatura nem de conectar contas de streaming**. Você monta sua lista de artistas e selos no próprio app, e os dados de lançamento vêm de fontes públicas de metadados musicais.

📐 **Plano visual completo (trains, features, wagons, interlockings, WMBTs, contratos, jornadas, telas e telemetria):** [Musicoven Plan](https://claude.ai/artifact/LESiQMUboHDJudKcfLiZuA)

> **Status:** plano decomposto e revisado (fase 2). Nada foi escrito em `plan/` ainda; a escrita dos YAMLs começa quando o plano for aprovado.

---

## Sumário

1. [O problema](#1-o-problema)
2. [Job to be Done](#2-job-to-be-done)
3. [Outcomes (WMBT) e metas](#3-outcomes-wmbt-e-metas)
4. [Princípios do produto](#4-princípios-do-produto)
5. [Decomposição ATDD](#5-decomposição-atdd)
6. [Telas sugeridas](#6-telas-sugeridas)
7. [Plano de telemetria](#7-plano-de-telemetria)
8. [Fontes de dados](#8-fontes-de-dados)
9. [Stack e decisões em aberto (ADRs)](#9-stack-e-decisões-em-aberto-adrs)
10. [Roadmap](#10-roadmap)
11. [Questões em aberto](#11-questões-em-aberto)

---

## 1. O problema

Quem acompanha muitos artistas perde lançamentos com frequência, porque:

- as funções de "novidades" dos serviços de streaming são ligadas a uma assinatura específica e deixam de funcionar quando você troca de serviço;
- as notificações desses serviços são pouco confiáveis: alguns lançamentos de artistas seguidos simplesmente não aparecem;
- os apps alternativos costumam exigir que você conecte suas contas de streaming;
- nenhuma dessas soluções guarda um **histórico** fácil de consultar ("o que saiu em março de 2024?", "quais discos dos meus artistas saíram em 2025?").

## 2. Job to be Done

> **Stay aware of album releases from the artists and labels I follow, and keep them organized by week, month and year so I remember what came out and when.**
>
> *(Estar ciente sobre os lançamentos de álbuns dos artistas e selos que sigo e organizar essa informação por semana, mês e ano para lembrar o que foi lançado e quando.)*

| Momento | Pergunta do usuário |
|---|---|
| **Descoberta e aviso** (presente/futuro) | "Algum artista ou selo que eu sigo lançou ou vai lançar um álbum?" |
| **Memória** (passado) | "O que foi lançado nesta semana / neste mês / neste ano?" |

## 3. Outcomes (WMBT) e metas

Cada outcome virou pelo menos um WMBT (*What Must Be True*), posicionado no wagon que o controla. As **metas são propostas** e precisam da sua validação.

| # | Outcome | Meta proposta | WMBTs |
|---|---|---|---|
| U1 | Minimizar a probabilidade de perder lançamentos de álbuns de artistas que sigo | ≥ 95% dos álbuns de artistas seguidos detectados (auditoria mensal) | `fetch-releases:M001`, `fetch-releases:R001`, `classify-releases:E001` |
| U2 | Maximizar a probabilidade de ficar sabendo quando os álbuns de artistas que gosto são lançados | ≥ 99% das notificações de lançamento entregues | `notify-releases:E001` |
| U3 | Minimizar o tempo para verificar quais álbuns foram lançados semanalmente | verificação semanal < 2 min | `classify-releases:E001`, `show-releases:L001` |
| U4 | Reduzir a frequência de perder lançamentos por não ficar sabendo | ≤ 1 lançamento por mês ainda não visto 7 dias após sair | `show-releases:M001`, `notify-releases:R001` |
| U5 | Aumentar a probabilidade de ser notificado na data do lançamento | ≥ 90% notificados na data do lançamento (horário local) | `notify-releases:E002`, `schedule-release-events:P001` |
| U6 | Minimizar a probabilidade de não conseguir ouvir o álbum na data do lançamento | ≥ 90% dos lançamentos abrem no serviço preferido no dia | `link-streams:E001` |
| U7 | Aumentar a probabilidade de saber quais álbuns foram lançados por ano | 100% dos lançamentos detectados mantidos; qualquer ano listado em < 30 s | `classify-releases:K001`, `show-releases:L002` |
| U8 | Minimizar o tempo para saber quais álbuns foram lançados por período (semana e ano) | trocar semana ↔ mês ↔ ano em < 1 s | `show-releases:Y001` |
| U9 | Minimizar o tempo para ver tudo o que é novo dos artistas seguidos | tudo o que é novo desde a última visita revisado em < 2 min | `show-releases:M001` |
| U10 | Minimizar o esforço para montar a lista de artistas seguidos | primeiros 20 seguidos em < 10 min | `sync-library:K001`, `resolve-artists:L001`, `manage-follows:D001` |
| U11 | Aumentar a probabilidade de saber de lançamentos futuros antes de saírem | ≥ 70% dos álbuns conhecidos ≥ 7 dias antes do lançamento | `classify-releases:M001`, `schedule-release-events:P001` |
| U12 | Minimizar a probabilidade de esquecer um lançamento salvo para depois | ≥ 80% dos favoritos abertos em até 30 dias | `sync-library:K001`, `mark-releases:M001` |
| U13 | Minimizar a probabilidade de perder um lançamento de selos seguidos | ≥ 95% dos lançamentos de selos seguidos detectados | `resolve-artists:L002`, `fetch-releases:L001` |

## 4. Princípios do produto

1. **Independente de streaming.** A lista de artistas e selos pertence ao usuário. Nenhum login de streaming é exigido; o login do musicoven serve só para sincronizar a biblioteca entre dispositivos.
2. **Cobertura acima de tudo.** Várias fontes são cruzadas e as duplicatas removidas, para não perder lançamentos (U1, U13).
3. **No dia certo.** O aviso chega na data do lançamento, no fuso do usuário, e antes dela quando o álbum é anunciado (U5, U11).
4. **Do aviso ao play em um toque.** Cada notificação e cada lançamento abre no serviço de streaming preferido (U6).
5. **O catálogo é permanente.** Todo lançamento detectado fica registrado por semana, mês e ano, mesmo após deixar de seguir o artista (U7, U8).
6. **Silêncio nunca esconde falha.** O app mostra quando verificou as fontes pela última vez e marca o que não foi notificado (U4).

## 5. Decomposição ATDD

A decomposição segue as convenções de planejamento do [`afokapu/atdd-bun`](https://github.com/afokapu/atdd-bun) (`planner-nodes/` e `planner-schemas/`), em três passes:

1. **Sistêmico:** job principal, outcomes, mapa das 9 etapas do job, temas e o grafo de artefatos entre wagons.
2. **Sistemático:** cada artefato escrito conforme o schema: trains, wagons, features, WMBTs, acceptances, contratos, interlockings, jornadas, telas e telemetria.
3. **Adversarial:** um script verifica as regras contra o plano (nomes verbo-objeto, ciclos, cobertura, categorias de rota, residuais, telemetria) e revisa a stack.

**Tamanho:** 14 trains · 11 wagons · 17 features · 23 WMBTs · 30 acceptances · 27 contratos · 4 interlockings · 4 jornadas · 11 telas · 25 itens de telemetria.

### 5.1 Temas e subjects

| Índice | Tema | Escopo |
|---|---|---|
| 0 | `commons` | Sincronização e backup da biblioteca entre dispositivos |
| 1 | `account` | Login: quem é o usuário em cada dispositivo |
| 2 | `artist` | Quais artistas e selos são seguidos |
| 3 | `release` | Coletar, classificar (selo, artista, álbum, ano), listar e abrir lançamentos |
| 4 | `alert` | Avisar sobre lançamentos: notificações e eventos de calendário |

| Subject | Significado |
|---|---|
| `session` | A sessão do usuário num dispositivo |
| `follow-list` | Os artistas e selos seguidos |
| `release` | Um lançamento de artista ou selo seguido |
| `feed` | A linha do tempo e o catálogo que o usuário navega |

### 5.2 Wagons e features

| Wagon | Tema | Responsabilidade | Features |
|---|---|---|---|
| `authenticate-listener` | account | Entra com um link de uso único enviado por e-mail, para que a mesma biblioteca acompanhe o usuário em todos os dispositivos. | `authenticate-magic-link` (**adicionada**) |
| `sync-library` | commons | Mantém o servidor como fonte da verdade e reaplica mudanças feitas offline num dispositivo. | `sync-account-library` (**adaptada**) |
| `resolve-artists` | artist | Encontra artistas e selos pelo nome nas fontes (MusicBrainz primeiro) e devolve uma identidade para cada. | `search-catalog-artists`<br>`search-record-labels` |
| `manage-follows` | artist | Mantém a lista de artistas e selos seguidos e recusa duplicatas. | `manage-artist-follows`<br>`manage-label-follows` |
| `fetch-releases` | release | Consulta as fontes para cada artista e selo seguido, em horários agendados e dentro dos limites de taxa, e registra indisponibilidades. | `poll-release-catalog` |
| `classify-releases` | release | Transforma candidatos em um lançamento cada, classificado por selo, artista, tipo (álbum, EP, single, ao vivo…) e ano, já lançado ou futuro, e guarda para sempre. | `classify-release-kinds`<br>`track-upcoming-releases` |
| `show-releases` | release | A linha do tempo: novidades desde a última visita, esta semana, próximos e o catálogo por semana, mês e ano, com filtros por tipo, artista e selo. | `show-release-timeline`<br>`filter-release-kinds`<br>`filter-artists-labels`<br>`partition-release-timeline` |
| `link-streams` | release | Abre o lançamento no serviço de streaming preferido, ou avisa que ainda não está disponível. | `open-streaming-app` (**adicionada**) |
| `mark-releases` | release | Salva lançamentos para ouvir depois. | `mark-listen-later` |
| `notify-releases` | alert | Envia push (ntfy) quando um artista ou selo seguido lança ou anuncia um álbum, uma vez por lançamento e tipo, na hora local configurada; marca como não visto no app se o push falhar. | `notify-new-releases` |
| `schedule-release-events` | alert | Adiciona um lançamento futuro ao calendário (evento .ics). | `schedule-calendar-events` |

Diferenças em relação às features escolhidas:
- **adicionada:** `authenticate-listener:authenticate-magic-link`, necessária para o tema Login;
- **adicionada:** `link-streams:open-streaming-app`, porque nada na seleção atende o U6 (ouvir no dia do lançamento). Confirmar se fica;
- **adaptada:** `sync-icloud-library` virou `sync-account-library`, porque uma PWA não acessa o iCloud.

### 5.3 Trains

| Train | Categoria | Origem | O que acontece |
|---|---|---|---|
| `train:session:sign-in` | nominal | adicionado | O usuário entra num dispositivo e recebe a biblioteca. |
| `train:session:resume-offline` | alternativo | adicionado | Um dispositivo volta a ficar online e os favoritos feitos offline são sincronizados. |
| `train:follow-list:add-artist-manually` | nominal | pedido | O usuário busca um artista e passa a segui-lo. |
| `train:follow-list:add-label-manually` | alternativo | pedido | O usuário busca um selo e passa a segui-lo. |
| `train:follow-list:refuse-duplicate-follow` | exceção | adicionado | Seguir algo que já é seguido é recusado. |
| `train:release:detect-new` | nominal | pedido | Um lançamento sai (novo, ou um anunciado cujo dia chegou) e o usuário é notificado. |
| `train:release:detect-upcoming` | alternativo | pedido | Um álbum é anunciado: o usuário recebe o anúncio e ele aparece em Próximos. |
| `train:release:refuse-duplicate-alert` | exceção | adicionado | Um lançamento já notificado é visto de novo e não gera segundo push. |
| `train:release:tolerate-outage` | erro | adicionado | Uma fonte está fora do ar; as outras rodam e a janela é repetida. |
| `train:release:flag-undelivered` | erro | adicionado | O ntfy está inacessível; o lançamento é marcado como não visto no app. |
| `train:feed:browse-timeline` | nominal | adicionado | O usuário abre a linha do tempo ou o catálogo e ouve um lançamento. |
| `train:feed:bookmark-release` | alternativo | pedido | O usuário salva um lançamento; ele aparece em Ouvir depois. |
| `train:feed:add-to-calendar` | alternativo | pedido | O usuário adiciona um lançamento futuro ao calendário. |
| `train:feed:streaming-unavailable` | erro | adicionado | O lançamento ainda não está no serviço de streaming. |

Os 6 trains pedidos foram mantidos. Os demais foram adicionados para que cada interlocking tenha uma rota nominal e trate as categorias de erro e exceção.

### 5.4 Interlockings e jornadas

| Interlocking | Ações de entrada | Rotas | Categorias sem rota |
|---|---|---|---|
| `interlocking:session` | `sign_in`, `resume_online` | `sign-in` → `train:session:sign-in` (nominal)<br>`resume` → `train:session:resume-offline` (alternativo) | exceção: residual `residual:expired-sign-in-link`<br>erro: residual `residual:sign-in-mail-outage` |
| `interlocking:follow` | `add_artist`, `add_label` | `artist` → `train:follow-list:add-artist-manually` (nominal)<br>`label` → `train:follow-list:add-label-manually` (alternativo)<br>`duplicate` → `train:follow-list:refuse-duplicate-follow` (exceção) | erro: residual `residual:search-source-outage` |
| `interlocking:release-scan` | `scan_releases` | `new` → `train:release:detect-new` (nominal)<br>`upcoming` → `train:release:detect-upcoming` (alternativo)<br>`duplicate` → `train:release:refuse-duplicate-alert` (exceção)<br>`outage` → `train:release:tolerate-outage` (erro)<br>`undelivered` → `train:release:flag-undelivered` (erro) | — |
| `interlocking:timeline` | `open_view`, `bookmark_release`, `add_to_calendar`, `listen` | `browse` → `train:feed:browse-timeline` (nominal)<br>`bookmark` → `train:feed:bookmark-release` (alternativo)<br>`calendar` → `train:feed:add-to-calendar` (alternativo)<br>`unavailable` → `train:feed:streaming-unavailable` (erro) | exceção: residual `residual:untracked-period` |

| Jornada | Entrada | Continuações |
|---|---|---|
| `journey:sign-in` | `interlocking:session` | — |
| `journey:follow-artists` | `interlocking:follow` | rota `artist` → `artist:follows` → `interlocking:release-scan`<br>rota `label` → `artist:follows` → `interlocking:release-scan` |
| `journey:stay-current` | `interlocking:release-scan` | — |
| `journey:browse-releases` | `interlocking:timeline` | — |

Seguir um artista ou selo dispara imediatamente uma varredura de lançamentos (continuação `artist:follows` → `interlocking:release-scan`).

### 5.5 Contratos

Cada artefato tem identidade `tema:aspecto[.variante]` e contrato em `contracts/<tema>/.../<aspecto>.schema.json`.

| Contrato | Produzido por | Consumido por |
|---|---|---|
| `account:refusals` | `wagon:authenticate-listener` | `user:listener` |
| `account:session` | `wagon:authenticate-listener` | `wagon:sync-library` |
| `account:sign-in-request` | entrada do usuário/sistema | `wagon:authenticate-listener` |
| `alert:calendar-events` | `wagon:schedule-release-events` | `user:listener` |
| `alert:deliveries` | `wagon:notify-releases` | `user:listener` |
| `alert:duplicates` | `wagon:notify-releases` | `system:monitor` |
| `alert:undelivered` | `wagon:notify-releases` | `wagon:show-releases` |
| `artist:duplicate-follows` | `wagon:manage-follows` | `user:listener` |
| `artist:follows` | `wagon:manage-follows` | `wagon:fetch-releases`, `wagon:notify-releases`, `user:listener` |
| `artist:identities` | `wagon:resolve-artists` | `wagon:manage-follows` |
| `artist:label-query` | entrada do usuário/sistema | `wagon:resolve-artists` |
| `artist:query` | entrada do usuário/sistema | `wagon:resolve-artists` |
| `commons:backup` | `wagon:sync-library` | `wagon:manage-follows`, `wagon:mark-releases` |
| `commons:offline-changes` | entrada do usuário/sistema | `wagon:sync-library` |
| `release:availability-gap` | `wagon:link-streams` | `user:listener` |
| `release:candidates` | `wagon:fetch-releases` | `wagon:classify-releases` |
| `release:entries` | `wagon:classify-releases` | `wagon:show-releases`, `wagon:notify-releases` |
| `release:entries.upcoming` | `wagon:classify-releases` | `wagon:show-releases`, `wagon:notify-releases` |
| `release:feed` | `wagon:show-releases` | `user:listener` |
| `release:feed-request` | entrada do usuário/sistema | `wagon:show-releases` |
| `release:fetch-gaps` | `wagon:fetch-releases` | `wagon:classify-releases` |
| `release:links` | `wagon:link-streams` | `user:listener` |
| `release:mark-request` | entrada do usuário/sistema | `wagon:mark-releases` |
| `release:marks` | `wagon:mark-releases` | `wagon:show-releases`, `user:listener` |
| `release:retry-schedule` | `wagon:classify-releases` | `system:scheduler` |
| `release:scan-request` | entrada do usuário/sistema | `wagon:fetch-releases` |
| `release:selection` | `wagon:show-releases` | `wagon:link-streams`, `wagon:schedule-release-events` |

## 6. Telas sugeridas

| Tela | Rota | Para quê | Outcomes |
|---|---|---|---|
| S01 · Entrar | `/entrar` | Entrar com um link enviado por e-mail. | — |
| S02 · Primeiros artistas | `/comecar` | Montar a lista de seguidos na primeira sessão. | U10 |
| S03 · Buscar | `/buscar` | Encontrar artistas e selos e segui-los. | U10, U13 |
| S04 · Seguindo | `/seguindo` | Ver e editar artistas e selos seguidos. | U10, U13 |
| S05 · Novidades | `/` | Início: tudo o que é novo desde a última visita e nesta semana, com filtros e status da última verificação. | U3, U9, U4, U1 |
| S06 · Próximos | `/proximos` | Álbuns anunciados com contagem regressiva; adicionar ao calendário. | U11, U5 |
| S07 · Catálogo | `/catalogo/2025` | Lançamentos por semana, mês e ano; grade de capas e contagem por mês. | U7, U8 |
| S08 · Lançamento | `/lancamento/:id` | Um lançamento: dados, Ouvir, salvar, calendário. | U6, U12 |
| S09 · Ouvir depois | `/ouvir-depois` | Lançamentos salvos, os mais antigos primeiro. | U12 |
| S10 · Ajustes | `/ajustes` | Notificações (tópico ntfy + teste), hora, fuso, serviço preferido, dispositivos. | U2, U5 |
| S11 · Notificação push | `ntfy` | O próprio push: o que chega na tela de bloqueio. | U2, U5 |

Os wireframes de baixa fidelidade estão no [plano visual](https://claude.ai/artifact/LESiQMUboHDJudKcfLiZuA#screens).

## 7. Plano de telemetria

Cada item segue o schema de tracking plan do atdd-bun: um arquivo por item em `telemetry/<tema>/<artefato>/<tipo>.<plano>[.<medida>].json`, com dono (o wagon que produz o artefato), acceptances que o exigem, dimensões sem alta cardinalidade e propriedades proibidas (e-mail, tópico do ntfy, tokens).

| Outcome | Como medir | Itens |
|---|---|---|
| U1 | ≥ 95% dos álbuns de artistas seguidos detectados (auditoria mensal) | `event:be:release:candidates`<br>`metric:be:release:candidates:staleness`<br>`metric:be:release:candidates:success_rate`<br>`metric:be:release:fetch-gaps:count`<br>`event:be:release:entries` |
| U2 | ≥ 99% das notificações de lançamento entregues | `event:be:alert:deliveries`<br>`metric:be:alert:deliveries:success_rate` |
| U3 | verificação semanal < 2 min | `event:be:release:entries`<br>`event:ui:release:feed`<br>`metric:ux:release:feed:duration` |
| U4 | ≤ 1 lançamento por mês ainda não visto 7 dias após sair | `metric:be:alert:undelivered:count` |
| U5 | ≥ 90% notificados na data do lançamento (horário local) | `metric:be:release:candidates:staleness`<br>`event:be:alert:deliveries`<br>`metric:be:alert:deliveries:latency` |
| U6 | ≥ 90% dos lançamentos abrem no serviço preferido no dia | `event:ui:release:links`<br>`metric:be:release:availability-gap:count` |
| U7 | 100% dos lançamentos detectados mantidos; qualquer ano listado em < 30 s | `event:be:release:entries`<br>`metric:be:release:entries:count`<br>`event:ui:release:feed` |
| U8 | trocar semana ↔ mês ↔ ano em < 1 s | `metric:ui:release:feed:latency` |
| U9 | tudo o que é novo desde a última visita revisado em < 2 min | `event:ui:release:feed`<br>`metric:ux:release:feed:duration` |
| U10 | primeiros 20 seguidos em < 10 min | `metric:be:commons:backup:success_rate`<br>`event:ui:artist:identities`<br>`event:ui:artist:follows`<br>`metric:ux:artist:follows:duration` |
| U11 | ≥ 70% dos álbuns conhecidos ≥ 7 dias antes do lançamento | `metric:be:release:entries:age`<br>`event:ui:alert:calendar-events` |
| U12 | ≥ 80% dos favoritos abertos em até 30 dias | `metric:be:commons:backup:success_rate`<br>`event:ui:release:marks`<br>`metric:be:release:marks:age` |
| U13 | ≥ 95% dos lançamentos de selos seguidos detectados | `event:ui:artist:identities`<br>`event:be:release:candidates`<br>`metric:be:release:candidates:success_rate` |

Armazenamento sugerido: uma tabela de eventos no mesmo banco, atrás de uma `TelemetryPort`, e uma página `/metrics` simples, sem SDK de terceiros.

## 8. Fontes de dados

Todas são públicas e nenhuma exige que o usuário conecte uma conta pessoal.

| Fonte | Uso | Observações |
|---|---|---|
| **[MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API)** | Fonte principal de artistas, **selos** (MBID) e *release groups* com tipo e data | Gratuita e aberta; ~1 req/s e User-Agent identificado |
| **[Deezer API](https://developers.deezer.com/api)** | Lançamentos recentes por artista, capas, datas e link para ouvir | Pública, sem autenticação para leitura |
| **[iTunes Search API](https://performance-partners.apple.com/search-api)** | Álbuns por artista no catálogo da Apple, inclusive pré-lançamentos | Gratuita, sem autenticação; tem limite de taxa |
| **[Odesli / song.link API](https://odesli.co)** | Links para os outros serviços de streaming | Base do "Ouvir no serviço preferido" |
| **[Cover Art Archive](https://coverartarchive.org)** | Capas ligadas ao MusicBrainz | Complementa Deezer/iTunes |
| **[AllMusic](https://www.allmusic.com)** | *New Releases* semanal, discografias, gêneros e estilos para conferir a cobertura | Sem API pública oficial: leitura das páginas com baixa frequência e cache, respeitando termos de uso e `robots.txt`; fonte complementar |

As fontes são detalhe de implementação: no plano elas ficam atrás de `fetch-releases:poll-release-catalog` e `resolve-artists`, e a escolha é registrada em ADR.

## 9. Stack e decisões em aberto (ADRs)

| Camada | Escolha | Por quê |
|---|---|---|
| Backend | Python + FastAPI | Simples, ótimo para integrar APIs e escrever jobs |
| Agendador | APScheduler (no próprio backend) | Evita infraestrutura extra; dá para trocar por cron depois |
| Banco | SQLite no início, Postgres se precisar | Zero configuração para uso pessoal |
| Frontend | PWA (React) | Instalável no celular e permite Web Push |
| Notificações | ntfy (push) | Gratuito |
| Hospedagem | Vercel | — |

A revisão adversarial encontrou conflitos entre a stack e os outcomes. Eles precisam de decisão antes da implementação:

1. **Vercel × APScheduler.** Na Vercel, o Python roda como função serverless: não existe processo contínuo, então o APScheduler não dispara. Isso contradiz o motivo "o app precisa ficar ligado para os jobs rodarem". O Vercel Cron pode chamar um endpoint de varredura, mas no plano Hobby roda no máximo uma vez por dia, o que é pouco para U1 e U5. Opções:
   - **A:** Vercel só para a PWA; FastAPI + APScheduler num host sempre ligado (Fly.io, Railway, Render).
   - **B:** tudo na Vercel, com Vercel Cron (plano Pro para execuções por hora) chamando `/jobs/scan` e `/jobs/notify`, sem APScheduler.
   - **C:** Vercel para o app e um workflow agendado do GitHub Actions chamando o endpoint de varredura a cada poucas horas.
2. **SQLite em serverless.** As funções da Vercel têm sistema de arquivos somente leitura (só `/tmp`, temporário), então o catálogo permanente (U7) se perderia. Use Postgres hospedado desde o início (Neon, Supabase, Vercel Postgres) ou libSQL/Turso. O SQLite só continua viável com a opção A (host com volume).
3. **Redundância de notificação.** A stack cita canais "redundantes", mas só lista o ntfy. Hoje o U4 depende do ntfy mais a marca de "não notificado" no app. Sugestão: Web Push da PWA como segundo canal.
4. **Telemetria.** Não há ferramenta na stack. Sugestão: tabela de eventos própria (seção 7).
5. **Login.** Link mágico por e-mail é a sugestão (passkeys são alternativa); ele exige um provedor de e-mail transacional.

## 10. Roadmap

### Fase 0: Fundação
- [ ] Decidir os ADRs da seção 9 (hospedagem, banco, notificação, telemetria, login)
- [ ] Escrever `plan/` (temas, subjects, wagons, features, WMBTs, trains, interlockings, jornadas) e `contracts/`, e ativar o `atdd-bun` (`planner`, `topology`, `traceability`, `telemetry`)
- [ ] Clientes MusicBrainz, Deezer e iTunes com controle de taxa

### Fase 1: Seguir e detectar (U1, U10, U13)
- [ ] `train:session:sign-in` e `train:session:resume-offline`
- [ ] `train:follow-list:add-artist-manually`, `add-label-manually`, `refuse-duplicate-follow`
- [ ] `train:release:tolerate-outage` e varredura agendada

### Fase 2: Avisar no dia (U2, U4, U5, U11)
- [ ] `train:release:detect-new`, `detect-upcoming`, `refuse-duplicate-alert`, `flag-undelivered`
- [ ] Ajustes de notificação (tópico ntfy, teste, hora, fuso)

### Fase 3: Ver e lembrar (U3, U6, U7, U8, U9, U12)
- [ ] `train:feed:browse-timeline`, `streaming-unavailable`, `bookmark-release`, `add-to-calendar`
- [ ] Catálogo por semana, mês e ano; Ouvir depois

### Fase 4: Medir
- [ ] Itens do plano de telemetria e página `/metrics` com as metas da seção 3
- [ ] Auditoria mensal de cobertura (U1, U13)

## 11. Questões em aberto

- Qual opção de hospedagem (A, B ou C) e qual banco?
- A feature `open-streaming-app`, adicionada para cobrir o U6, fica?
- Que hora local deve sair o push do dia de lançamento?
- Quais tipos de lançamento notificam por padrão (só álbum e EP?)
- Qual o serviço de streaming preferido padrão?
- Seguir um artista novo deve trazer lançamentos antigos para o catálogo? Até quantos anos?
- As metas propostas na seção 3 estão certas?
- O U10 (esforço para montar a lista) depende só de busca manual. Importar de uma biblioteca ou do Last.fm reduziria muito esse esforço: entra no escopo?
