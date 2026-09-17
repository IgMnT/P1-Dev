# Plataforma de Processamento de Vídeo

Upload de vídeo sob demanda: conversão para 360p/720p/1080p, extração de áudio
para legendagem automática e marca d'água, com progresso em tempo real no
navegador. Nenhuma ação roda automaticamente — o usuário escolhe, por
upload, exatamente quais resoluções converter, se extrai áudio e se aplica
marca d'água (ver `domain/processingActions.js`).

Além do fluxo de processamento, o projeto usa esta base para demonstrar, na
prática, dois temas de engenharia de sistemas distribuídos: **consistência
em transações distribuídas** (Transactional Outbox, para resolver o
problema de Dual-Write entre Postgres e RabbitMQ) e **cache/estado
distribuído com Redis** (Cache-Aside, mitigação de Cache Penetration e
Cache Stampede, rate limiting e política de evicção). Ver a seção
[ADR](#adr--decisões-de-arquitetura) para o racional de cada decisão.

## Requisitos

- Docker e Docker Compose

## Como rodar

```bash
docker compose up -d --build
```

Aguarde os serviços ficarem `healthy`/`running` (`docker compose ps`) e abra
`http://localhost:3000` no navegador para usar o mini-cliente de upload, ou
teste via API. Uma [collection do Postman](./postman_collection.json) com
todos os endpoints (incluindo os payloads de exemplo) também está disponível
na raiz do projeto.

O upload aceita quais ações executar via campos de formulário
(`resolutions` pode se repetir, uma vez por resolução marcada):

```bash
curl -F "video=@caminho/do/video.mp4" \
     -F "resolutions=720p" \
     -F "resolutions=1080p" \
     -F "extractAudio=true" \
     -F "watermark=true" \
     http://localhost:3000/videos
# => {
#      "videoId": "...",
#      "status": "uploaded",
#      "actions": {"resolutions":["720p","1080p"],"extractAudio":true,"watermark":true},
#      "destination": {
#        "directory": "/app/storage",
#        "files": ["<nome>-720p.mp4", "<nome>-1080p.mp4", "<nome>.mp3"]
#      }
#    }

curl http://localhost:3000/videos/<videoId>
# => {"id":"...","originalFilename":"video.mp4","storagePath":"...","status":"processing", ...}
# 1ª chamada: cache miss no Redis, resposta vem do Postgres (e fica cacheada).
# Chamadas seguintes dentro do TTL: cache hit, sem tocar o banco.

curl -N http://localhost:3000/videos/<videoId>/progress
# stream de eventos SSE: {"stage":"convertendo_720p","percent":100,"outputPath":"<nome>-720p.mp4",...}

curl -O -J http://localhost:3000/videos/<videoId>/files/<nome>-720p.mp4
```

Pelo menos uma ação precisa ser selecionada (uma resolução ou a extração de
áudio) — sem isso a API responde `400` (`ProcessingActions`, em
`src/domain/processingActions.js`). O campo `destination` avisa, já na
resposta do upload, onde os arquivos resultantes vão ser gravados no volume
compartilhado; cada evento SSE de conclusão de etapa repete o `outputPath`
daquele arquivo específico, e o mini-cliente usa isso para virar um link de
download (`GET /videos/:id/files/:filename`) assim que o arquivo fica
pronto. Esse endpoint só libera nomes que batem com a convenção de saída do
próprio vídeo pedido (`usecases/getVideoFile.js`) — path traversal ou nomes
de outro vídeo voltam `400`/`404`.

`POST /videos` tem rate limiting por IP (Redis, `INCR`+`EXPIRE`): mais de 20
uploads em 60s do mesmo IP (configurável via `RATE_LIMIT_MAX` /
`RATE_LIMIT_WINDOW_SECONDS`) retornam `429`:

```bash
for i in $(seq 1 25); do curl -s -o /dev/null -w "%{http_code}\n" -F "video=@v.mp4" -F "resolutions=360p" http://localhost:3000/videos; done
# ...várias linhas "202", depois "429" a partir da 21ª dentro da mesma janela
```

Para demonstrar o escalonamento horizontal dos workers (competing consumers):

```bash
docker compose up -d --scale worker=3
```

### Portas expostas no host

RabbitMQ e Redis foram remapeados (`5673`, `15673`, `6380`) para evitar
conflito com outros serviços que já pudessem estar rodando na máquina de
desenvolvimento. As portas internas dos containers continuam as padrão
(`5672`, `15672`, `6379`) — ajuste `docker-compose.yml` livremente se não
houver conflito no seu ambiente.

| Serviço | Host | Container |
|---|---|---|
| API | 3000 | 3000 |
| Postgres | 5432 | 5432 |
| RabbitMQ (AMQP) | 5673 | 5672 |
| RabbitMQ (painel) | 15673 | 15672 |
| Redis | 6380 | 6379 |

`outbox-relay` e `worker` não expõem porta nenhuma — só consomem Postgres,
Redis e/ou RabbitMQ internamente na rede do compose.

## Testes

```bash
npm install
npm test
```

Testes automatizados (`node:test`, sem framework adicional) cobrem:

- `domain/`: invariantes de `Video`, parsing de `ProcessingJob`, validação de
  `processingActions` (pelo menos uma ação escolhida, marca d'água sem
  resolução é desligada), arredondamento/`outputPath` de `ProgressUpdate` e a
  lista de `resolutions`.
- `usecases/`: `uploadVideo` (grava vídeo + job atomicamente via
  `videoRepository.saveWithJob`, sem conhecer outbox/AMQP), `processVideo`
  (execução seletiva das ações, falha do ffmpeg, falha best-effort da
  transcrição), `getVideoFile` (convenção de nomes de download),
  `getVideoStatus` (cache hit, cache miss + populamento do cache, mitigação
  de Cache Penetration via marcador de "não encontrado") e
  `dispatchOutboxEvent` (publica evento pendente, no-op sem eventos, erro do
  publisher propaga para a transação fazer rollback) — todos com mocks
  manuais das dependências injetadas.
- `interfaces/messaging/videoJobConsumer`: ack em sucesso, nack sem requeue em
  JSON inválido e em erro do usecase.
- `interfaces/http/progressController`: assinatura/desinscrição do canal
  Redis por vídeo, incluindo o caso de duas conexões SSE assistindo ao mesmo
  vídeo (a primeira que fecha não pode cortar a segunda).
- `interfaces/http/rateLimitMiddleware`: libera dentro do limite, bloqueia
  com `429` acima do limite, chaveia por IP e encaminha erro do Redis para o
  error handler.

Fluxos de integração (upload → outbox → relay → fila → worker → SSE) foram
validados manualmente ponta a ponta durante o desenvolvimento, subindo os
containers reais e enviando um vídeo gerado com `ffmpeg` (ver histórico de
commits).

## Limitação conhecida: legendagem automática

`TRANSCRIPTION_API_URL` aponta para `http://transcription.invalid/v1` — um
domínio do TLD reservado `.invalid` (RFC 2606), que nunca resolve por design.
Isso simula uma API externa de transcrição sem exigir uma integração real
para o trabalho acadêmico. A chamada em `services/transcriptionService.js` é
best-effort: se falhar, o worker registra um aviso e o vídeo é concluído
normalmente, sem legenda. Para usar um serviço real, basta apontar a
variável de ambiente para o endpoint correto.

## Arquitetura

Clean Architecture com direção de dependência `interfaces → usecases →
domain`; `infra` e `services` implementam o que os usecases consomem via
injeção de dependência manual (`src/container.js`). Ver `CLAUDE.md` para a
descrição completa da estrutura de pastas e das regras de camada.

Três processos compartilham o mesmo código-fonte/imagem Docker, diferindo só
pelo `command` (ver `docker-compose.yml`):

| Processo | Entrypoint | Responsabilidade |
|---|---|---|
| `api` | `src/server.js` | HTTP: upload, status (cache-aside), SSE de progresso, download |
| `worker` | `src/worker.js` | Consome a fila AMQP, roda o ffmpeg, publica progresso no Redis |
| `outbox-relay` | `src/outboxRelay.js` | Drena a tabela `outbox` e publica os jobs pendentes na fila AMQP |

## Modelagem de latência: por que cachear o status do vídeo

`GET /videos/:id` é o candidato natural a cache: é consultado repetidamente
pelo mesmo cliente durante o processamento (o mini-cliente e qualquer
integração fariam polling dele se não houvesse SSE) e a resposta muda pouco
entre uma chamada e outra. Aplicando a fórmula de latência efetiva:

```
L_efetiva = (HR × L_RAM) + ((1 − HR) × L_BD)
```

Com Redis na mesma rede do compose (`L_RAM ≈ 1 ms`) e uma consulta indexada
por chave primária no Postgres (`L_BD ≈ 5 ms`, adotando um valor conservador
para uma tabela pequena com índice de PK), um Hit Ratio de 90% já reduz a
latência efetiva de 5 ms (sem cache) para `(0,90 × 1) + (0,10 × 5) = 1,4 ms`
— uma redução de 72% na latência típica de leitura, sem tirar carga alguma
do Postgres nas leituras restantes cacheadas. O TTL curto (30s + jitter,
`CACHE_TTL_SECONDS`/`CACHE_JITTER_SECONDS`) mantém o Hit Ratio alto durante
o processamento (múltiplas leituras de status por vídeo em poucos minutos)
sem arriscar servir um `status: "completed"` desatualizado por muito tempo
depois do processamento realmente terminar.

## ADR — Decisões de arquitetura

Formato: Contexto → Decisão → Consequências. Alternativas descartadas ficam
ao final de cada bloco temático.

### Comunicação entre processos

#### ADR-01 — Fila de trabalho: AMQP (RabbitMQ)

**Contexto:** a conversão de vídeo é uma operação longa (minutos) e cara em
CPU; a API que recebe o upload não pode ser a mesma que processa, e o
processamento precisa sobreviver à morte de um worker no meio do trabalho.

**Decisão:** usar uma fila durável (`video-processing`) com ack manual e
`prefetch(1)` para distribuir os jobs de conversão entre a API e os workers.

**Consequências:** uma fila durável com ack garante que o job não se perde
se um worker morrer no meio do processamento — a mensagem volta para a fila
em vez de desaparecer. O padrão *competing consumers* permite escalonamento
horizontal simplesmente subindo mais réplicas de worker
(`docker compose up -d --scale worker=3`), sem nenhuma mudança de código.
Em troca, a API depende de outro serviço externo (RabbitMQ) para o fluxo de
upload funcionar — mitigado pela decisão ADR-04 (Outbox), que remove essa
dependência síncrona.

#### ADR-02 — Progresso worker → API: Redis Pub/Sub

**Contexto:** o worker precisa informar o percentual real do ffmpeg à API
enquanto processa, para a API repassar ao navegador.

**Decisão:** o worker publica cada evento de progresso em um canal Redis por
vídeo (`video:<id>`); a API assina esse canal para repassar aos clientes
conectados via SSE.

**Consequências:** progresso é informação efêmera — uma atualização
desatualizada não tem valor, então não há necessidade de persistência ou
garantia de entrega, ao contrário do job em si. Pub/Sub é fan-out puro e
desacopla completamente a API do número de workers rodando: nenhum dos dois
lados precisa saber quantas réplicas existem do outro. O custo é que quem
conecta depois do evento ser publicado nunca o recebe (sem replay) — ver
"Armadilhas conhecidas".

#### ADR-03 — Navegador: SSE (Server-Sent Events)

**Contexto:** o navegador precisa receber progresso em tempo real sem dar
refresh, num fluxo estritamente unidirecional (servidor → cliente).

**Decisão:** expor o progresso via `GET /videos/:id/progress` usando SSE
nativo do Express, sem biblioteca adicional.

**Consequências:** `EventSource` roda sobre HTTP comum, tem reconexão
automática embutida no navegador e não exige nenhuma infraestrutura de
proxy/load balancer diferenciada, ao contrário de WebSocket. Em troca, o
canal é só de leitura — se o produto precisasse de comandos do cliente para
o servidor no mesmo canal (ex.: cancelar o processamento), SSE não bastaria.

### Consistência de dados distribuída

#### ADR-04 — Transactional Outbox para o job de upload

**Contexto:** a versão inicial deste projeto fazia exatamente o antipadrão
descrito no material de Transações em Sistemas Distribuídos: `uploadVideo`
gravava o vídeo no Postgres e, em seguida, publicava a mensagem no RabbitMQ
como dois passos separados (**Dual-Write**). Postgres e RabbitMQ são
sistemas independentes, sem uma transação distribuída nativa entre eles — se
o `INSERT` fosse commitado e a publicação no AMQP falhasse logo depois
(RabbitMQ fora do ar, timeout de rede), o vídeo ficaria salvo como
`uploaded` para sempre, sem nenhum worker jamais ser acionado, e sem
nenhum log de erro visível ao usuário. Um `try/catch` ao redor das duas
chamadas não resolve nada, porque não existe rollback do outro lado.

**Decisão:** aplicar o padrão **Transactional Outbox**. `uploadVideo`
grava o vídeo e o payload do job na tabela `outbox` (`db/init.sql`) dentro
da mesma transação ACID do Postgres
(`postgresVideoRepository.saveWithJob`) — ou os dois `INSERT`s commitam
juntos, ou nenhum dos dois. Um processo dedicado, o **outbox relay**
(`src/outboxRelay.js`, um *Polling Publisher* simples — a opção
recomendada pelo material para um projeto deste porte, em vez de CDC via
Debezium), consulta a tabela em loop (`OUTBOX_POLL_INTERVAL_MS`, padrão
500ms), publica cada evento pendente na fila e só marca `published_at` após
a confirmação da publicação (garantia *at-least-once*, com deduplicação
natural porque o consumer já é idempotente por design — reprocessar um job
de conversão não corrompe o resultado final). A leitura usa
`SELECT ... FOR UPDATE SKIP LOCKED` para não publicar o mesmo evento duas
vezes caso o relay seja escalado no futuro.

**Consequências:**
- A API deixou de depender do RabbitMQ para aceitar um upload: se o broker
  cair, o Postgres continua aceitando novos vídeos (ficam pendentes na
  outbox até o RabbitMQ voltar) em vez de a requisição falhar — uma
  aplicação prática do princípio *Basically Available* do modelo BASE.
- `uploadVideo` (usecase) não sabe que existe uma tabela `outbox` ou um
  broker AMQP por trás — só chama `videoRepository.saveWithJob(video, job)`,
  preservando a regra de dependência da Clean Architecture.
- Custo: mais uma tabela para gerenciar (crescimento não-limpo é mitigado
  pelo índice parcial `WHERE published_at IS NULL`, que mantém a
  verificação de pendências barata mesmo com o histórico crescendo) e mais
  um processo para operar (`outbox-relay`), além de uma latência adicional
  de até um ciclo de polling (500ms) entre o upload responder `202` e o job
  efetivamente entrar na fila — imperceptível frente aos minutos que a
  própria conversão leva.

#### ADR-05 — Por que não Two-Phase Commit (2PC) nem uma Saga completa

**Contexto:** o material de Transações em Sistemas Distribuídos apresenta
2PC e o padrão Saga (coreografia/orquestração) como alternativas para
coordenar múltiplos serviços/bancos.

**Decisão:** não usar nenhum dos dois aqui — nem para o problema
Postgres↔RabbitMQ (resolvido com Outbox, ADR-04), nem para o fluxo de
processamento do vídeo em si.

**Por quê:**
- **2PC** exigiria um coordenador bloqueante segurando locks no Postgres e
  no RabbitMQ durante toda a fase de preparação — exatamente o tipo de
  acoplamento síncrono que este sistema foi desenhado para evitar (a
  conversão dura minutos; nada aqui pode ficar bloqueado esperando).
  Também não existe suporte nativo a 2PC entre PostgreSQL e RabbitMQ sem
  bibliotecas/infra adicionais, e o ganho (consistência forte imediata) não
  compensa o risco de o coordenador travar o sistema inteiro numa partição
  de rede.
- **Saga** coordena uma cadeia de transações locais *em múltiplos serviços
  de negócio diferentes* (ex.: Pedido → Pagamento → Estoque → Entrega, cada
  um compensável). Este projeto tem um único domínio de negócio (processar
  um vídeo) executado por réplicas *idênticas* de um mesmo worker — não há
  múltiplos serviços com efeitos colaterais de negócio distintos para
  coordenar nem compensar. O único ponto real de dual-write é a fronteira
  Postgres↔fila, que o Outbox já resolve de forma muito mais simples. Se o
  domínio crescesse para incluir, por exemplo, cobrança por conversão e
  notificação por e-mail como serviços separados, aí sim uma Saga (provável
  orquestração, dado o baixo número de serviços e a necessidade de
  auditoria de billing) passaria a se justificar.

**Consequências:** o sistema aceita consistência eventual apenas na janela
entre o `INSERT` na outbox e a publicação pelo relay (tipicamente < 1s) —
compatível com o modelo BASE, já que "vídeo aceito mas ainda não
enfileirado" é um estado transitório sem impacto observável pelo usuário
(a resposta HTTP já é `202 Accepted`, não `200 OK`).

#### ADR-06 — CAP: por que Redis e RabbitMQ aqui priorizam AP

**Contexto:** o Teorema CAP força uma escolha entre Consistência e
Disponibilidade sob partição de rede.

**Decisão:** tanto o Redis (cache de status e Pub/Sub de progresso) quanto o
fluxo de upload (via Outbox) são tratados como **AP** — sob falha parcial,
o sistema prefere responder com dado potencialmente desatualizado (ou
aceitar o upload mesmo com o broker fora do ar) a recusar a requisição.

**Consequências:** é aceitável porque nenhum destes dados é financeiro nem
irreversível — um `status` de vídeo com até `CACHE_TTL_SECONDS` de atraso,
ou um evento de progresso perdido por falta de assinante no momento exato,
não corrompe nada, só atrasa a percepção do usuário. Se este sistema um dia
processasse cobrança por vídeo convertido, essa escolha precisaria ser
revisitada para CP naquele fluxo específico (ex.: validação de saldo antes
de aceitar o upload).

### Cache e estado com Redis

#### ADR-07 — Cache-Aside para `GET /videos/:id`, com mitigação de Cache Penetration e Cache Stampede

**Contexto:** consultar o status de um vídeo é uma leitura simples e
repetida (mesmo padrão de acesso de um dashboard), boa candidata a cache; e
o Redis já está no `docker-compose.yml` para o Pub/Sub de progresso (ADR-02).

**Decisão:** `usecases/getVideoStatus.js` implementa Cache-Aside puro:
consulta o Redis (`cache:video:<id>`) primeiro; em miss, busca no
`videoRepository` (Postgres), devolve ao cliente e popula o cache. Duas
mitigações vêm junto:
- **Cache Penetration**: se o vídeo não existe no Postgres, o resultado
  "não encontrado" também é cacheado (`__NOT_FOUND__`, TTL curto —
  `CACHE_NOT_FOUND_TTL_SECONDS`), para que tentativas repetidas com um
  `videoId` inválido (por engano ou abuso) não gerem uma consulta ao banco
  a cada chamada.
- **Cache Stampede**: o TTL de cada entrada recebe um jitter aleatório
  (`CACHE_JITTER_SECONDS`) somado ao TTL base, para que vídeos cacheados no
  mesmo instante não expirem todos juntos e gerem um pico simultâneo de
  leituras no Postgres.

**Consequências:** o usecase permanece limpo — recebe `videoRepository` e
`videoCache` por parâmetro, sem saber que "cache" aqui é Redis
(`infra/redisVideoCache.js` implementa o `get`/`set`/`setNotFound` com
Strings + `EX`). Fica como extensão documentada, mas não implementada, um
lock distribuído (`SET NX EX`, estilo Redlock) para o caso avançado de
"apenas um requisitante recarrega o cache enquanto os demais aguardam" — o
volume de tráfego deste projeto não justifica essa complexidade adicional
além do jitter, mas o padrão está descrito no material de referência caso o
sistema precise escalar.

#### ADR-08 — Rate limiting com Redis Strings (INCR + EXPIRE)

**Contexto:** `POST /videos` é uma operação cara (grava arquivo em disco,
abre transação, enfileira um job de processamento) — um cliente
malicioso ou um bug de retry sem backoff no front-end poderia enfileirar
uploads mais rápido do que os workers conseguem processar.

**Decisão:** middleware `rateLimitMiddleware` + `infra/redisRateLimiter.js`
implementam uma janela fixa por IP: `INCR ratelimit:upload:<ip>`, e só no
primeiro hit da janela (`count === 1`) arma o `EXPIRE` (`RATE_LIMIT_WINDOW_SECONDS`).
Acima de `RATE_LIMIT_MAX` requisições na janela, a API responde `429` antes
mesmo de aceitar o multipart do arquivo.

**Consequências:** `INCR` é atômico no Redis mesmo com múltiplas réplicas da
API concorrendo na mesma chave (não precisa de lock manual). O contador
convive na mesma instância Redis do cache/Pub/Sub, cada um com seu próprio
namespace de chave (`ratelimit:`, `cache:video:`, `video:` para os canais).

#### ADR-09 — Política de evicção do Redis: `allkeys-lru`

**Contexto:** com `maxmemory` configurado, o Redis precisa de uma política
de evicção quando o limite é atingido.

**Decisão:** `allkeys-lru` (`docker-compose.yml`, serviço `redis`), com
`maxmemory 128mb`.

**Consequências:** todas as chaves que este projeto guarda no Redis — cache
de status (TTL curto), contadores de rate limit (TTL curto) e canais de
Pub/Sub (sem estado persistente) — se beneficiam de "remover primeiro o que
está inativo há mais tempo", a mesma lógica recomendada no material para
sessões de usuário: chamadas recentes (vídeo em processamento agora) devem
sobreviver mais que chamadas antigas (vídeo consultado uma vez há uma
hora). Não há dado crítico armazenado sem TTL no Redis deste projeto, então
não existe risco de perder informação que não possa ser reconstruída a
partir do Postgres.

### Alternativas descartadas

- **gRPC** para a comunicação entre serviços: otimiza RPC síncrono de baixa
  latência. O caso de uso aqui é o oposto — uma operação que dura minutos e
  exige desacoplamento temporal entre quem pede e quem processa. Um request
  gRPC bloqueado por minutos não é uma boa modelagem do problema.
- **WebSocket** para o navegador: o cliente só recebe atualizações, nunca
  envia nada pelo mesmo canal. A bidirecionalidade do WebSocket traria
  handshake próprio, gestão de estado de conexão e complexidade adicional de
  proxy/load balancer sem nenhuma contrapartida real para este fluxo.
- **Redis como fila de trabalho** (em vez de AMQP, para os jobs de
  conversão): Pub/Sub não tem ack nem persistência de mensagem — um worker
  que morre no meio de uma conversão de 10 minutos perderia o job em
  silêncio, sem nenhuma forma de retry ou redistribuição. Isso é aceitável
  para *progresso* (efêmero por natureza), mas inaceitável para o *job em
  si*, que representa trabalho real do usuário.
- **2PC** entre Postgres e RabbitMQ (em vez de Outbox): ver ADR-05.
- **Saga completa** (coreografia ou orquestração) para o pipeline de
  processamento: ver ADR-05 — não há múltiplos serviços de negócio
  distintos para coordenar/compensar neste domínio.
- **CDC (Debezium) em vez de Polling Publisher** para o outbox relay: o
  material recomenda CDC para sistemas de alto throughput com requisito de
  baixíssima latência; este projeto não tem esse volume, e o Polling
  Publisher já entrega latência (~500ms) irrelevante frente aos minutos de
  conversão, sem exigir infraestrutura adicional (Kafka Connect/Debezium).
- **Lock distribuído (Redlock)** para Cache Stampede em `GET /videos/:id`:
  ver ADR-07 — o jitter no TTL já é suficiente para o volume de tráfego
  deste projeto; o lock fica documentado como extensão, não implementado.

## Armadilhas conhecidas

Ver seção "Armadilhas conhecidas" em `CLAUDE.md` — volume compartilhado
API/worker, cabeçalhos SSE corretos (sem `compression()`), `prefetch(1)` no
worker, progresso real do ffmpeg (não sintético). Além delas, específicas
desta entrega:

- **Três instâncias de `ioredis`, não duas.** A API já precisava de uma
  conexão em modo subscriber (SSE) separada de uma conexão de publish
  (worker) — mas o subscriber não aceita nenhum outro comando enquanto
  assina um canal. Com Cache-Aside e rate limiting, a API agora abre uma
  **terceira** conexão (`infra/redisClient.js`), de propósito geral
  (`GET`/`SET`/`INCR`/`EXPIRE`), nunca compartilhada com o subscriber.
- **`outbox-relay` não tem `container_name` por acidente, tem por decisão
  contrária ao `worker`.** Diferente do worker (que é seguro escalar
  livremente, `docker compose up -d --scale worker=3`), o relay roda
  como instância única de propósito — o volume de uploads deste projeto não
  justifica escalá-lo, embora o uso de `FOR UPDATE SKIP LOCKED` já deixasse
  seguro fazer isso se um dia fosse necessário.
- **A outbox cresce sem limite se nunca for limpa.** Em produção real, o
  relay (ou um job separado) precisaria arquivar/apagar linhas com
  `published_at` antigo. Fora do escopo desta entrega acadêmica, mas
  registrado aqui de propósito (o próprio material de referência chama
  atenção para esse ponto).
