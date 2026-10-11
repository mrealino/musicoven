# 🎶 musicoven

> Saiba quando seus artistas favoritos lançam um álbum, no dia em que ele sai, e guarde tudo num catálogo organizado por mês e ano.

O **musicoven** é uma aplicação pessoal que acompanha os lançamentos de álbuns dos artistas que você segue, avisa no dia do lançamento (ou antes, se o lançamento já tiver sido anunciado) e mantém um histórico consultável do que saiu em cada semana, mês e ano.

Ele **não depende de uma assinatura nem de conectar contas de streaming**. Você monta sua lista de artistas diretamente no app, e os dados de lançamento vêm de fontes públicas de metadados musicais.

---

## Sumário

1. [O problema](#1-o-problema)
2. [Job to be Done](#2-job-to-be-done)
3. [Outcomes desejados](#3-outcomes-desejados)
4. [Princípios do produto](#4-princípios-do-produto)
5. [Funcionalidades](#5-funcionalidades)
6. [Mapa de outcomes → funcionalidades](#6-mapa-de-outcomes--funcionalidades)
7. [Fluxos principais](#7-fluxos-principais)
8. [Fontes de dados](#8-fontes-de-dados)
9. [Arquitetura proposta](#9-arquitetura-proposta)
10. [Modelo de dados](#10-modelo-de-dados)
11. [Roadmap](#11-roadmap)
12. [Métricas de sucesso](#12-métricas-de-sucesso)
13. [Riscos e mitigações](#13-riscos-e-mitigações)
14. [Questões em aberto](#14-questões-em-aberto)

---

## 1. O problema

Quem acompanha muitos artistas perde lançamentos com frequência, porque:

- as funções de "novidades" dos serviços de streaming são ligadas a uma assinatura específica e deixam de funcionar quando você troca de serviço;
- as notificações desses serviços são pouco confiáveis: alguns lançamentos de artistas seguidos simplesmente não aparecem;
- os apps alternativos costumam exigir que você conecte suas contas de streaming;
- nenhuma dessas soluções guarda um **histórico** fácil de consultar ("o que saiu em março de 2024?", "quais discos dos meus artistas saíram em 2025?").

## 2. Job to be Done

> **Estar ciente sobre os lançamentos de álbuns de artistas que gosto e organizar essa informação por mês e ano para lembrar o que foi lançado e quando, ao longo do tempo.**

O job se divide em dois momentos:

| Momento | Pergunta do usuário |
|---|---|
| **Descoberta e aviso** (presente/futuro) | "Algum artista que eu sigo lançou ou vai lançar um álbum?" |
| **Memória** (passado) | "O que foi lançado nesta semana / neste mês / neste ano?" |

## 3. Outcomes desejados

| # | Outcome | Direção |
|---|---|---|
| O1 | Probabilidade de perder lançamentos de álbuns de artistas que gosto | ⬇️ minimizar |
| O2 | Probabilidade de ficar sabendo quando os álbuns de artistas que gosto são lançados | ⬆️ maximizar |
| O3 | Tempo para verificar quais álbuns foram lançados semanalmente | ⬇️ minimizar |
| O4 | Frequência de perder lançamentos por não ficar sabendo | ⬇️ reduzir |
| O5 | Probabilidade de ser notificado na data do lançamento | ⬆️ aumentar |
| O6 | Probabilidade de não conseguir ouvir o álbum na data do lançamento | ⬇️ minimizar |
| O7 | Probabilidade de saber quais álbuns foram lançados por ano | ⬆️ aumentar |
| O8 | Tempo para saber quais álbuns foram lançados por período (semana e ano) | ⬇️ minimizar |

## 4. Princípios do produto

1. **Independente de streaming.** A lista de artistas pertence ao usuário e não a uma plataforma. Nenhum login de streaming é obrigatório.
2. **Cobertura acima de tudo.** É melhor cruzar várias fontes e remover duplicatas do que depender de uma só e perder um lançamento (O1, O4).
3. **No dia certo.** O aviso precisa chegar na data do lançamento, e antes dela quando o lançamento já foi anunciado (O5).
4. **Do aviso ao play em um toque.** Cada notificação já leva aos links do álbum em vários serviços de streaming (O6).
5. **O catálogo é permanente.** Todo lançamento detectado fica registrado e pode ser consultado por semana, mês e ano (O7, O8).
6. **Pouco atrito.** Um resumo semanal deve bastar para quem não quer abrir o app todo dia (O3).

## 5. Funcionalidades

### 5.1 Gestão de artistas seguidos
- **Busca e follow de artistas** por nome, com desambiguação (foto, país, gêneros, álbuns conhecidos) para não seguir o artista homônimo errado.
- **Importação em massa (opcional)**, sem conectar contas:
  - colar uma lista de nomes (um por linha);
  - importar um arquivo CSV/TXT;
  - importar uma biblioteca exportada (ex.: arquivo de biblioteca do Apple Music/iTunes, histórico do Last.fm por nome de usuário público).
- **Sugestões de artistas** relacionados aos que você já segue (fase posterior).
- **Preferências por artista**: tipos de lançamento acompanhados (álbum, EP, single, ao vivo, coletânea, deluxe/reedição) e prioridade de notificação (avisar sempre / só no resumo).

### 5.2 Detecção de lançamentos
- **Job agendado** (ex.: a cada 6 horas) que consulta as fontes de dados para cada artista seguido.
- **Cruzamento de múltiplas fontes** com deduplicação: o mesmo álbum vindo de fontes diferentes vira um único registro.
- **Lançamentos futuros (anunciados)**: quando a fonte informa uma data futura, o álbum entra como "anunciado" e passa para "lançado" na data.
- **Filtro por tipo**: por padrão só álbuns de estúdio e EPs; singles, coletâneas e reedições são opcionais.
- **Atualização de datas**: se a data de um lançamento anunciado mudar, o registro é atualizado e o usuário é avisado.

### 5.3 Notificações
- **No dia do lançamento**: "🎉 *Artista* lançou *Álbum* hoje", com capa e links para ouvir.
- **Lançamento anunciado**: aviso quando um novo álbum é anunciado, com a data prevista.
- **Lembrete na véspera** (opcional) para lançamentos anunciados.
- **Resumo semanal** (ex.: sexta-feira, quando a maioria dos lançamentos sai): todos os álbuns da semana numa única mensagem.
- **Canais** (configuráveis, mais de um ao mesmo tempo para redundância):
  - push no celular (ex.: [ntfy](https://ntfy.sh) ou Web Push/PWA);
  - bot no Telegram;
  - e-mail.
- **Feed de calendário (.ics)**: lançamentos anunciados aparecem no calendário do usuário (Google Calendar, Apple Calendar etc.).
- **Feed RSS** dos lançamentos, como alternativa passiva.

### 5.4 Ouvir no dia
- **Links universais** para o álbum em vários serviços (Apple Music, Deezer, YouTube Music, Tidal, Bandcamp, Spotify…), gerados automaticamente.
- **Serviço preferido**: o usuário escolhe o serviço principal e o botão "Ouvir" da notificação abre direto nele.
- **Status "disponível"**: indicar se o álbum já aparece nos serviços (alguns lançamentos chegam com atraso em certas plataformas).

### 5.5 Catálogo / histórico
- **Linha do tempo** de lançamentos, organizada por **ano → mês → semana**.
- **Visões rápidas**: "Esta semana", "Este mês", "Este ano", "Próximos lançamentos".
- **Página do ano**: todos os álbuns lançados no ano pelos artistas seguidos, com capas em grade, contagem por mês e filtros por artista, gênero e tipo.
- **Página do artista**: discografia detectada pelo app, em ordem cronológica.
- **Busca e filtros**: por artista, título, gênero, tipo e período.
- **Marcações pessoais** (fase posterior): "ouvido", "quero ouvir", nota e favoritos, para virar também um diário musical.
- **Exportação** do catálogo (CSV/JSON) para o usuário não ficar preso ao app.

### 5.6 Configurações
- Fuso horário e dia/horário do resumo semanal.
- Canais de notificação e serviço de streaming preferido.
- Tipos de lançamento acompanhados (padrão global + exceções por artista).

## 6. Mapa de outcomes → funcionalidades

| Outcome | Funcionalidades que atendem |
|---|---|
| **O1** Minimizar perder lançamentos | Cruzamento de fontes; job agendado frequente; importação em massa de artistas; desambiguação de artistas |
| **O2** Maximizar ficar sabendo | Notificação no dia; aviso de anúncio; múltiplos canais; feed .ics e RSS |
| **O3** Minimizar tempo de verificação semanal | Resumo semanal; visão "Esta semana" |
| **O4** Reduzir perdas por não ficar sabendo | Canais redundantes; resumo semanal como rede de segurança; feed .ics |
| **O5** Notificar na data do lançamento | Detecção de lançamentos anunciados; agendamento do aviso para a data; lembrete na véspera |
| **O6** Conseguir ouvir na data | Links universais; serviço preferido; status de disponibilidade |
| **O7** Saber o que saiu por ano | Catálogo permanente; página do ano; exportação |
| **O8** Minimizar tempo para consultar por período | Linha do tempo ano → mês → semana; visões rápidas; filtros |

## 7. Fluxos principais

### Fluxo 1: Onboarding
1. O usuário abre o app e define o fuso horário.
2. Adiciona artistas por busca, colando uma lista ou importando um arquivo.
3. Para nomes ambíguos, escolhe o artista correto numa lista com foto e contexto.
4. Escolhe os canais de notificação e o serviço de streaming preferido.
5. O app faz uma **varredura inicial** e preenche o catálogo com os lançamentos recentes (ex.: últimos 12 meses) e os próximos anunciados.

### Fluxo 2: Detecção → notificação
```
[Agendador] → para cada artista seguido → consulta fontes
      → normaliza e remove duplicatas → compara com o catálogo
      → novo lançamento?
           ├─ data futura   → salva como "anunciado" → notifica anúncio → agenda aviso para a data
           └─ data ≤ hoje   → salva como "lançado"   → busca links de streaming → notifica
```

### Fluxo 3: Resumo semanal
1. No dia/horário configurado, o app reúne os lançamentos dos últimos 7 dias.
2. Envia uma mensagem única com capas, artistas e links.
3. A mensagem tem um link para a visão "Esta semana" do catálogo.

### Fluxo 4: Consulta ao catálogo
1. O usuário abre o catálogo, que mostra o ano corrente por padrão.
2. Navega por mês ou semana, ou pula para outro ano.
3. Filtra por artista/gênero/tipo e abre o álbum para ver detalhes e links.

## 8. Fontes de dados

Todas são públicas e nenhuma exige que o usuário conecte uma conta pessoal.

| Fonte | Uso | Observações |
|---|---|---|
| **[MusicBrainz API](https://musicbrainz.org/doc/MusicBrainz_API)** | Fonte principal de artistas (IDs estáveis/MBID) e *release groups* com tipo (álbum, EP, single) e data | Gratuita e aberta; limite de ~1 req/s e exige User-Agent identificado; às vezes a data de lançamento demora a ser cadastrada |
| **[Deezer API](https://developers.deezer.com/api)** | Lançamentos recentes por artista, capas, datas e link para ouvir | Pública, sem autenticação para leitura |
| **[iTunes Search API](https://performance-partners.apple.com/search-api)** | Álbuns por artista no catálogo da Apple, inclusive pré-lançamentos | Gratuita, sem autenticação; tem limite de taxa |
| **[Odesli / song.link API](https://odesli.co)** | A partir de um link, gera links para os outros serviços de streaming | Base para os "links universais" |
| **[Cover Art Archive](https://coverartarchive.org)** | Capas de álbuns ligadas ao MusicBrainz | Complementa as capas do Deezer/iTunes |
| **[AllMusic](https://www.allmusic.com)** | Página de *New Releases* semanal, discografias, gêneros/estilos e resenhas para enriquecer o catálogo e servir de conferência da cobertura | Não tem API pública oficial: o acesso seria por leitura das páginas, respeitando os termos de uso e o `robots.txt`, com baixa frequência e cache. Por isso, entra como fonte complementar e de auditoria, não como fonte principal |
| **Last.fm API** (opcional) | Importar artistas mais ouvidos de um perfil público e sugerir artistas relacionados | Usa só o nome de usuário público, sem login |

**Estratégia de cruzamento:** o MusicBrainz é a "identidade" do artista. Para cada artista guardamos os IDs equivalentes no Deezer e no iTunes. A cada varredura, as três fontes são consultadas, e os álbuns são unidos por *(artista, título normalizado, data ± tolerância)*.

O AllMusic entra como camada extra: a lista semanal de *New Releases* é comparada com os artistas seguidos para encontrar lançamentos que as outras fontes ainda não trouxeram (O1, O4). Os gêneros e estilos do AllMusic também enriquecem os filtros do catálogo.

## 9. Arquitetura proposta

Como é uma aplicação pessoal, a recomendação é uma arquitetura **simples e barata de manter**: um único backend com agendador embutido e um banco de dados leve.

```
┌──────────────────┐     ┌───────────────────────────────┐     ┌────────────────────┐
│  Frontend (PWA)  │◄───►│  API (backend)                │◄───►│  Banco de dados    │
│  catálogo, busca │     │  artistas, catálogo, config   │     │  SQLite → Postgres │
│  configurações   │     ├───────────────────────────────┤     └────────────────────┘
└──────────────────┘     │  Agendador (jobs)             │
                         │  • varredura de lançamentos   │──► MusicBrainz / Deezer / iTunes / AllMusic
                         │  • avisos do dia / véspera    │──► Odesli (links)
                         │  • resumo semanal             │──► ntfy / Telegram / e-mail
                         └───────────────────────────────┘
                               │
                               └──► feeds .ics e RSS
```

**Stack sugerida** (pode ser revista antes da fase 1):

| Camada | Escolha | Por quê |
|---|---|---|
| Backend | Python + FastAPI | Simples, ótimo para integrar APIs e escrever jobs |
| Agendador | APScheduler (no próprio backend) | Evita infraestrutura extra; dá para trocar por cron depois |
| Banco | SQLite no início, Postgres se precisar | Zero configuração para uso pessoal |
| Frontend | PWA (ex.: React/Vite ou HTMX) | Instalável no celular e permite Web Push |
| Notificações | ntfy (push) + Telegram bot + e-mail | Canais gratuitos e redundantes |
| Hospedagem | VPS pequena, Fly.io/Railway ou Raspberry Pi em casa | O app precisa ficar ligado para os jobs rodarem |

## 10. Modelo de dados

```
Artist
  id, name, sort_name, mbid, deezer_id, itunes_id, image_url, country, genres

Follow
  artist_id, release_types[], notify_mode (always | digest_only), followed_at

Release
  id, artist_id, title, normalized_title, type (album | ep | single | live | compilation | reissue),
  release_date, date_precision (day | month | year), status (announced | released),
  cover_url, genres, source_ids {mbid, deezer_id, itunes_id}, first_seen_at, updated_at

ReleaseLink
  release_id, service (apple_music | deezer | spotify | youtube_music | tidal | bandcamp …), url

Notification
  id, release_id, kind (announced | release_day | eve | date_changed | weekly_digest),
  channel, sent_at, status

UserEntry   (fase posterior)
  release_id, listened (bool), rating, favorite, notes

Settings
  timezone, digest_weekday, digest_time, channels{}, preferred_service, default_release_types[]
```

As visões por **semana / mês / ano** são consultas sobre `Release.release_date`. Não há tabelas separadas para elas.

## 11. Roadmap

### Fase 0: Fundação
- [ ] Definir a stack final e criar a estrutura do repositório
- [ ] Clientes para MusicBrainz, Deezer e iTunes, respeitando os limites de taxa
- [ ] Avaliar os termos de uso do AllMusic e definir como ler as páginas de *New Releases* e de discografia
- [ ] Esquema do banco de dados e migrações

### Fase 1: MVP "nunca mais perder um lançamento" (O1, O2, O5)
- [ ] Busca de artistas com desambiguação e follow/unfollow
- [ ] Importação de lista de artistas (colar texto / CSV)
- [ ] Job de varredura com cruzamento de fontes e deduplicação
- [ ] Detecção de lançamentos anunciados e lançados
- [ ] Notificação no dia do lançamento por um canal (ntfy ou Telegram)

### Fase 2: Ouvir no dia + resumo semanal (O3, O4, O6)
- [ ] Links universais via Odesli e serviço preferido
- [ ] Resumo semanal configurável
- [ ] Aviso de anúncio e lembrete na véspera
- [ ] Canais adicionais (e-mail) e feed .ics

### Fase 3: Catálogo (O7, O8)
- [ ] Linha do tempo ano → mês → semana
- [ ] Visões rápidas (semana, mês, ano, próximos)
- [ ] Página do ano e página do artista
- [ ] Busca, filtros e exportação CSV/JSON

### Fase 4: Refinamentos
- [ ] PWA instalável com Web Push
- [ ] Marcações pessoais (ouvido, nota, favorito)
- [ ] Importação via Last.fm e sugestões de artistas relacionados
- [ ] Feed RSS
- [ ] Estatísticas anuais ("seu ano em lançamentos")

## 12. Métricas de sucesso

| Outcome | Métrica | Meta inicial |
|---|---|---|
| O1, O4 | % de lançamentos de artistas seguidos que o app detectou (auditoria manual mensal) | ≥ 95% |
| O2, O5 | % de lançamentos notificados até o fim do dia do lançamento | ≥ 90% |
| O5 | Atraso médio entre o lançamento e a notificação | < 12 h |
| O6 | % de notificações com link funcional para o serviço preferido | ≥ 90% |
| O3 | Tempo para revisar o resumo semanal | < 2 min |
| O7, O8 | Tempo para responder "o que saiu em [mês/ano]?" no catálogo | < 30 s |

## 13. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Uma fonte demora a cadastrar um lançamento | Cruzar várias fontes; resumo semanal como rede de segurança |
| Artistas homônimos geram falsos positivos | Desambiguação no follow; guardar IDs por fonte |
| Duplicatas (deluxe, remaster, edições regionais) | Normalização de títulos; agrupar por *release group*; filtro de reedições |
| Limites de taxa das APIs | Fila com controle de taxa, cache e varreduras escalonadas |
| Datas imprecisas (só ano ou mês) | Campo `date_precision`; notificar quando a data exata for confirmada |
| Mudanças ou descontinuação de APIs externas | Camada de "provedores" isolada para trocar uma fonte sem mexer no resto |
| O servidor fica fora do ar e perde a janela do aviso | Ao reiniciar, o job recupera e envia avisos pendentes; monitoramento simples (healthcheck) |

## 14. Questões em aberto

- Uso **só pessoal** ou com suporte a vários usuários no futuro? Isso afeta autenticação e modelo de dados.
- Quais **canais de notificação** priorizar no MVP: push (ntfy), Telegram ou e-mail?
- Singles devem notificar por padrão ou só aparecer no catálogo?
- Onde hospedar: nuvem gerenciada ou servidor em casa?
- Qual **serviço de streaming principal** usar como padrão nos links?

---

*Este plano será refinado com a análise de funcionalidades e fluxos de uma plataforma de referência.*
