### HLD: n3x-controller (host de plugins, servidor gRPC e painel)

Versão: 1.1
Data: 2026-10-20
Responsável: Guilherme Giusti

---

### Objetivo técnico
Descrever o controller do N3X como um **host de plugins sem armazenamento próprio**: um processo Go que hospeda as metades de controller dos plugins e oferece a eles três coisas. A primeira é um servidor gRPC único (`17443`, TLS 1.3), em que a autenticação é feita pelo autenticador do plugin core e a autorização pelo host. A segunda é um painel em Echo v5 + templ + htmx (`17444`). A terceira é métricas e health (`17091`). O host cuida da sessão do painel, do root, da autorização, da contenção de falhas e do ciclo de vida dos plugins. **Quem persiste dados é sempre o plugin**, com a própria conexão de banco (ADR-002). Usuários de leitura, inventário e configs ficam no plugin core (ADR-003), em memória na v1 (ADR-001 do `n3x-plugin-core`); a auditoria formal fica para a v2 (ADR-002 do `n3x-plugin-core`).

Este HLD é dono destes contratos (conforme `docs/hld/HLDs-N3X.md`):
- Schema do `n3x.yaml` do controller
- Servidor gRPC: convenção `n3x.plugins.<namespace>.<nome>.v1`, códigos de erro, identidade do chamador e limites de conexões, streams e mensagens
- Comportamento do host de plugins: ordem de subida, obrigatório e opcional, contenção de panic e estado por plugin
- Painel: layout, sessão, papéis por rota, slots de telas e cartões e atualização ao vivo
- Armazenamento: decisão de que o host **não** tem armazenamento (ADR-002)

Problemas técnicos endereçados
- O PRD deixou o armazenamento do controller em aberto; a decisão é tirar a persistência do host e deixá-la com cada plugin
- O host precisa isolar os plugins (namespace gRPC, rotas, capacidades) e conter panics em telas e serviços
- A meta é 500 sidecars conectados em 1 vCPU e 512 MB, com o painel em p95 < 300 ms
- O login é a única rota pública e vira alvo de força bruta e de DoS de CPU por bcrypt, inclusive na rajada após um restart
- O root nunca pode ficar no armazenamento, e o host precisa funcionar mesmo com o core `falho` ou `degradado`

Base: `docs/prd/PRD-N3X.md` versão 1.5, de 2026-10-20 (já atualizada com as decisões deste HLD e do HLD do `n3x-plugin-core`). Requisitos cobertos: FR-005, FR-008, FR-009 e FR-011. FR-010 (auditoria) foi adiado para a v2 (ADR-002 do `n3x-plugin-core`). FR-006 e FR-007 entram só no papel de host (a lógica e os dados são do plugin core). Também cobre os requisitos não funcionais de desempenho, disponibilidade, segurança, observabilidade e acessibilidade do controller. As metas do sistema estão divididas por componente conforme o ADR-010 do `n3x-sdk`.

Dependências com outros sistemas
- `n3x-sdk`: contrato de plugin de controller, incluindo as capacidades privilegiadas do core (só referenciado aqui)
- `n3x-plugin-core`: rota de login, autenticador de sessão, chave HMAC dos tokens, diretório de usuários (só leitura), inventário, configs e telas, tudo em memória na v1 (só referenciado aqui)
- `n3x-builder`: imagem `controller-builder` e `n3x-plugin build`, que geram o binário a partir do módulo `host/` deste repositório
- `n3x-sidecar`: cliente do servidor gRPC pela conexão compartilhada; cliente do mesmo formato `ENC[...]`, agora no pacote `secret` do `n3x-sdk` (ADR-014 do `n3x-sdk`)
- Orquestradores: Kubernetes, ECS e docker compose (topologias em `n3x-sdk/deploy/`)
- Prometheus, opcional

---

### Arquitetura geral
O controller é um processo Go único, em **instância única por ambiente** (ADR-001 do `n3x-sdk`), fora do caminho crítico do tráfego.

**Entregas do repositório** (ADR-001)
- **Módulo Go `host/`**: servidor gRPC, painel, sessão, host de plugins e operação. Não gera binário aqui: o `n3x-plugin build` cria um `main` que chama `host.Run(core.Plugin(), <plugins>...)` e compila o binário com as metades de controller escolhidas
- **Imagem base `ghcr.io/giustech/n3x/controller:<semver>`** (semver próprio, ADR-001 do `n3x-builder`): runtime sem binário, em amd64 e arm64 e formato OCI (hipótese: distroless static nonroot, fixada por digest)
- **`cmd/n3x-controller-dev`**: `main` mínimo só com o plugin core, para desenvolvimento local e CI

```
n3x-controller/
├── host/
│   ├── run.go                 host.Run e subcomandos
│   ├── config/                schema do n3x.yaml do controller v1
│   ├── plugins/               registro, dependências, ciclo de vida, estado
│   ├── grpcsrv/               servidor 17443 e interceptors
│   ├── credential/            verificador de credencial (bcrypt + cache)
│   ├── session/               sessão do painel, root, bloqueio de login
│   ├── panel/                 Echo v5, layout, menu, cartões, assets embutidos
│   └── ops/                   17091: /metrics, /live, /ready
├── cmd/n3x-controller-dev/    main de desenvolvimento (host + plugin core)
├── dev/                       n3x.yaml de desenvolvimento
├── images/                    Dockerfile da imagem base
└── test/load/                 simulador de 500 sidecars
```

**Topologia em runtime**
```
                        container do controller (não root)
  sidecars ──► :17443 ┌───────────────────────────────────────────────┐
  (gRPC, TLS 1.3)     │ grpcsrv: recover → log/métricas → limites     │
                      │          → autenticação → autorização          │
                      │          → serviços n3x.plugins.<namespace>.<nome>.v1        │
  operadores ─► :17444│ panel: recover → headers → sessão → CSRF       │
  (HTTPS)             │        → papel da rota → telas e cartões       │
  Prometheus ─► :17091│ ops: /metrics /live /ready                     │
  orquestrador        │                                                │
                      │ host/plugins ── plugin core ──────────────────►│──► memória do core (v1)
                      │              ── plugins opcionais ────────────►│──► banco de cada plugin
                      └───────────────────────────────────────────────┘
```

**Sequência de subida (`run`)**
1. Lê `/etc/n3x/controller.yaml` (ou `--config`) e valida o schema v1; campo desconhecido é erro
2. Lê o manifesto embutido no binário (seção ELF `.n3x_manifest`, ADR-003 do `n3x-builder`) e confere: integridade (SHA-256 das seções carregáveis), `format` conhecido, versão do `host/` com a mesma major e minor igual ou menor que a da imagem (fora disso, encerra dizendo qual versão usar), plugin core presente, IDs únicos, nenhuma dependência faltando ou em ciclo
3. Lê a chave AES do arquivo, decifra os `ENC[...]` e valida a seção de cada plugin em `plugins` contra o schema dele
4. Carrega o certificado TLS (a chave HMAC dos tokens é do plugin core, ADR-004 do `n3x-plugin-core`)
5. `Init` de cada plugin em ordem de dependência, com os handles do SDK: config tipada, logger, métricas, memória isolada, registrador gRPC, registrador de telas e cartões e canal de estado. As capacidades privilegiadas vão só para o core. Serviço fora de `n3x.plugins.<namespace>.<nome>.v1` ou rota fora de `/plugins/<namespace>/<nome>/` é recusado
6. `Start` de cada plugin com prazo (`runtime.plugin_start_timeout`, padrão de 30 s). Se um obrigatório falha, o controller encerra com código diferente de zero; se um opcional falha, ele fica no estado `falho`, sem serviços nem telas
7. Abre `17091` (`/live` já responde), depois `17443` e `17444`; `/ready` só responde 200 nesse ponto

**Desligamento (SIGTERM)**
1. `/ready` passa a 503
2. `GracefulStop` do gRPC e `Shutdown` do painel com prazo (`runtime.shutdown_timeout`, padrão de 20 s), seguido de parada forçada
3. `Stop` dos plugins em ordem inversa, com 5 s por plugin
4. Encerra

**Subcomandos** (ADR-009)
| Subcomando | Função |
| --- | --- |
| `run` | Sobe o controller (padrão) |
| `validate` | Valida o `n3x.yaml` e as seções dos plugins sem abrir portas |
| `hash` | Gera bcrypt (custo 12) da senha lida do stdin, para o root |
| `credential new <serviço>` | Gera um segredo aleatório de 32 bytes; imprime o hash para `services.<serviço>.secret_hash` e o segredo para cifrar com `n3x encrypt` no lado do sidecar |
| `encrypt` | Cifra em `ENC[...]` um valor lido do stdin (ex.: senha do banco de um plugin) |
| `keygen` | Gera 32 bytes aleatórios: a chave AES ou a `session.hmac_key` do core (cifrada depois com `encrypt`) |
| `dev-cert` | Gera um certificado autoassinado para desenvolvimento |

**Desenvolvimento local**
- `go.work` na pasta `n3x/` com `n3x-controller`, `n3x-plugin-core` e `n3x-sdk`, para editar os três sem publicar versão
- `n3x-controller dev-cert`, `templ generate --watch` e `go run ./cmd/n3x-controller-dev --config dev/n3x.yaml`
- O `dev/n3x.yaml` usa o plugin core em memória (único modo da v1), root com senha `dev` e um serviço de teste; o painel abre em `https://localhost:17444`
- Não há dependência circular: o plugin core importa só o SDK, nunca o `host/`

Ambiente de implantação
- Agnóstico: cloud ou on-premises, onde houver containers (topologias de referência em `n3x-sdk/deploy/`, ADR-008 do `n3x-sdk`)
- Uma instância por ambiente, com escala só vertical (ADR-001 do `n3x-sdk`)
- `n3x.yaml`, certificado TLS e chave AES montados como arquivos (ConfigMap e Secret no Kubernetes, volume no ECS e no docker compose)
- O banco de cada plugin é provisionado pelo operador, conforme o HLD daquele plugin

Tecnologias principais
- Go 1.26 ou superior, sem cgo, binário estático
- grpc-go e protobuf
- Echo v5 (5.4.0), templ e htmx, com htmx embutido no binário e servido localmente
- `golang.org/x/crypto/bcrypt`
- `log/slog` com saída JSON e `prometheus/client_golang`
- Pacote `secret` do `n3x-sdk` para `ENC[...]` (AES-256-GCM), ADR-014 do `n3x-sdk`
- axe nos testes (sem Testcontainers: o core da v1 não tem banco)

Padrões adotados
- Host de plugins com registro explícito e ordem por dependência
- Cadeia de interceptors no gRPC e de middlewares no painel
- Autorização no host, antes de qualquer handler de plugin
- Capacidades privilegiadas exclusivas do plugin core (ADR-004 do `n3x-sdk`)
- Host sem estado persistente: tudo que é durável pertence a um plugin (ADR-002)
- Falha rápida na subida e degradação graciosa por plugin
- Prazo em toda chamada a plugin

---

### Componentes e responsabilidades
| Componente | Responsabilidades | Dependências |
| ----------- | ----------------- | ------------ |
| CLI e `host.Run` | Subcomandos, exit codes e mensagens sem segredos; sequência de subida e desligamento | `host/config`, `secret` do `n3x-sdk` |
| Config (`host/config`) | Schema v1 do `n3x.yaml` do controller; decifra `ENC[...]`; valida a seção de cada plugin pelo schema dele | Manifesto embutido, chave AES |
| Host de plugins (`host/plugins`) | Registro; ordem por dependência; `Init`, `Start` e `Stop` com prazo; obrigatório ou opcional; checagem de namespace e de rotas; capacidades privilegiadas só para o core; estado por plugin (`ok`, `degradado`, `falho`) | SDK |
| Servidor gRPC (`host/grpcsrv`) | TLS 1.3 em `17443`; interceptors de recover, log e métricas, limites, autenticação (autenticador do core, exceto `Login`) e autorização (só papel `sidecar` na v1, ADR-007); identidade do chamador no `ctx`; mapeamento de erros | Autenticador do core |
| Verificador de credencial (`host/credential`) | Confere o segredo do serviço contra `services.<nome>.secret_hash`; limite de tentativas por IP e serviço; no máximo 2 bcrypt simultâneos; cache de 1 h de credenciais verificadas (ADR-004) | `n3x.yaml` |
| Sessão do painel (`host/session`) | Login e logout; root conferido pelo arquivo; usuários de leitura conferidos pelo diretório do core; sessão só em memória (ADR-006); cookie seguro; CSRF; bloqueio de 5 tentativas por 15 min; login, logout e bloqueio no log JSON operacional | Diretório de usuários do core |
| Painel (`host/panel`) | Echo v5 + templ + htmx em `17444`; layout, menu, login, visão geral montada com cartões dos plugins, banner de estado por plugin, slots `/plugins/<namespace>/<nome>/`; papel mínimo por rota; polling de 5 s (ADR-008); assets embutidos; WCAG 2.1 AA | Telas e cartões dos plugins |
| Operação (`host/ops`) | `17091`: `/metrics`, `/live` e `/ready` | Todos |
| Plugin core (outro repositório) | Rota `Login`, autenticador de sessão, chave HMAC, diretório de usuários só leitura, inventário, configs, telas de sidecars, configs, fluxo por serviço e lista de usuários, cartões da visão geral (ADR-003) | Memória própria na v1 (ADR-001 do `n3x-plugin-core`) |
| Plugins opcionais | Serviços gRPC, telas, cartões e o próprio banco, se precisarem | SDK, banco próprio |

Quem persiste dados: só os plugins, cada um com a própria conexão. Quem faz cache: o host guarda em memória sessões, contadores de bloqueio, o cache de credenciais e o estado dos plugins; cada plugin tem a memória isolada do SDK. Quem orquestra: o host de plugins (ciclo de vida) e as cadeias de interceptors e middlewares (requisições).

---

### Fluxo de requisições e de dados
**Fluxo de requisição: chamada gRPC de um sidecar**
- O sidecar conecta em `17443` com TLS 1.3, pela conexão compartilhada
- Interceptor de recover: um panic no handler vira `INTERNAL` com mensagem genérica, log com stack amostrado e `plugin_panics_total`
- Interceptor de log e métricas, com `request_id` vindo dos metadados
- Limites: conexões, streams por conexão e tamanho de mensagem; excedente recebe `RESOURCE_EXHAUSTED`
- Autenticação: `n3x.plugins.n3x.core.v1.Login` passa direto; os demais métodos têm o token conferido pelo autenticador do core, que devolve `{papel: sidecar, serviço, instância}` ou `UNAUTHENTICATED`
- Autorização: na v1, todo método exige o papel `sidecar` (ADR-007); outro papel recebe `PERMISSION_DENIED`
- O plugin dono do namespace recebe a chamada com a identidade no `ctx`; se ele estiver `falho` ou parando, a resposta é `UNAVAILABLE`

**Fluxo de requisição: login de um sidecar**
- O handler `Login` do core recebe serviço, instância e segredo
- O core chama o verificador de credencial do host:
  - IP ou serviço bloqueado → `RESOURCE_EXHAUSTED`
  - Credencial no cache de verificadas → aceita sem bcrypt
  - Senão, espera vaga entre as 2 verificações simultâneas (fila curta com prazo; sem vaga, `RESOURCE_EXHAUSTED`) e compara com o `secret_hash`
  - Falha → conta a tentativa e devolve `UNAUTHENTICATED`; sucesso → entra no cache
- O core emite o token HMAC (ADR-004 e ADR-005 do `n3x-plugin-core`)

**Fluxo de requisição: painel**
- O operador acessa `17444` com TLS 1.3
- Recover (panic em tela vira 500 com página genérica) → request id e log → headers de segurança (CSP, `X-Frame-Options`, `Referrer-Policy`)
- Sessão pelo cookie; sem sessão, redireciona ao login guardando a página de origem
- CSRF em todo método que não é GET
- Papel mínimo da rota (leitura por padrão; root para alterações e para a lista de usuários); sem permissão, 403 com mensagem clara
- Handler do host ou do plugin; a resposta é um pedaço de HTML se a chamada veio do htmx (`HX-Request`) ou a página inteira com layout
- Blocos de status com `hx-trigger="every 5s"` (intervalo em `panel.refresh_interval`)

**Fluxo de requisição: login no painel**
- Confere o bloqueio por usuário e IP (5 tentativas, 15 min)
- Usuário igual ao root: compara com o bcrypt do arquivo; senão, consulta o diretório do core
- Erro: mensagem genérica, sem revelar se o usuário existe
- Sucesso: cria a sessão em memória e devolve o cookie; login, logout e bloqueio vão para o log JSON operacional (não é auditoria formal, que fica para a v2, ADR-002 do `n3x-plugin-core`)
- Exceção de emergência: se o core estiver `falho` ou `degradado`, só o root entra

**Fluxo de dados**
- `n3x.yaml` + chave AES (arquivos) → `host/config` (valida e decifra) → config tipada de cada plugin
- Sidecar → gRPC → handler do core → memória do core → telas e cartões do core
- Login, logout e bloqueio no painel → log JSON operacional do host
- Usuários de leitura → seção do core no `n3x.yaml` → diretório só leitura do core; mudar usuário é editar o arquivo e reiniciar (ADR-003 do `n3x-plugin-core`)
- Estado de cada plugin → host → banner do painel e `n3x_controller_plugin_state`
- Logs JSON → stderr → coletor do orquestrador; métricas → `17091` → Prometheus

---

### Modelo de dados (alto nível)
Entidades principais
- **Config do controller** (`n3x.yaml` v1): schema abaixo
- **Root:** usuário e hash bcrypt, vindos só do arquivo
- **Credencial de serviço:** nome do serviço e um único `secret_hash` (ADR-005)
- **Sessão do painel** (memória): ID aleatório de 32 bytes, usuário, papel, criada em, último acesso e token CSRF
- **Contador de bloqueio** (memória): chave usuário+IP no painel e IP+serviço no login de sidecar, tentativas e bloqueado até
- **Credencial verificada** (memória): HMAC-SHA256 do segredo com chave aleatória do processo, serviço e expira em (1 h)
- **Identidade do chamador:** `{papel: root | leitura | sidecar, usuário ou serviço+instância}`
- **Estado de plugin** (memória): `ok`, `degradado` ou `falho`, com motivo e horário
- **Evento de acesso ao painel** (log JSON): login, logout e bloqueio, com usuário, IP e horário; a auditoria formal fica para a v2

**Schema do `n3x.yaml` do controller v1**
```yaml
version: 1

listen:
  grpc: 17443
  panel: 17444
  ops: 17091

tls:
  cert_file: /etc/n3x/tls.crt
  key_file: /etc/n3x/tls.key

keys:
  aes_file: /run/n3x/aes.key              # decifra os ENC[...] das seções de plugin
                                          # chave HMAC: session.hmac_key do core (ADR-004 do n3x-plugin-core)

root:
  user: admin
  password_hash: "$2b$12$..."             # n3x-controller hash

services:                                 # credenciais dos sidecars, só hashes
  pedidos:
    secret_hash: "$2b$12$..."             # n3x-controller credential new pedidos

grpc:
  limits:
    max_connections: 1000
    max_streams_per_connection: 16
    max_message: 4MiB
  login:
    max_failures: 5                       # por IP e por serviço
    lockout: 15m
    bcrypt_concurrency: 2
    verified_cache_ttl: 1h

panel:
  session:
    idle_timeout: 30m
    absolute_timeout: 12h
  login:
    max_failures: 5                       # por usuário e IP
    lockout: 15m
  refresh_interval: 5s

runtime:
  log_level: info                         # debug | info | warn | error
  shutdown_timeout: 20s
  plugin_start_timeout: 30s
  plugin_call_timeout: 10s                # teto por chamada unária gRPC e tela de plugin; streams: 1 s por mensagem

plugins:                                  # uma seção por plugin, validada pelo schema dele
  n3x.core:                               # schema no HLD do n3x-plugin-core
    inventory_retention: 24h
    memory_limit: 128MiB
    session:
      hmac_key: ENC[v1:...]               # opcional, recomendado em produção
      ttl: 1h
    users:                                # usuários de leitura, só pelo arquivo
      - name: ana
        password_hash: "$2a$12$..."
        disabled: false
```

Regras de validação (no `validate` e no `run`)
- `version` precisa ser `1`; campo desconhecido é erro
- `root.password_hash` e cada `secret_hash` precisam ser bcrypt com custo ≥ 12; sem root, o controller não sobe
- Nome de serviço único; nenhum usuário de leitura pode ter o nome do root (conferido pelo `Validate` do core na subida)
- Cada chave de `plugins` precisa existir no manifesto; a seção é validada pelo schema do plugin
- `ENC[...]` precisa decifrar; o erro cita o caminho do campo, nunca o valor
- Nenhum campo vem de variável de ambiente

Relações
- Controller 1:1 `n3x.yaml`; `n3x.yaml` 1:N credenciais de serviço; serviço 1:1 `secret_hash`
- Controller 1:N plugins; plugin 1:1 estado; plugin 0:1 banco próprio
- Usuário (root ou leitura) 1:N sessões do painel

Fonte de verdade
- `n3x.yaml` do controller: root, credenciais, portas, limites e config dos plugins; lido só na subida
- Manifesto embutido no binário: lista de plugins, versões e schemas
- Plugin core (em memória na v1): inventário e configs; usuários de leitura vêm da seção do core no `n3x.yaml`
- Cada plugin: os próprios dados; um plugin acessa dados de outro só pela API pública dele

Versionamento e retenção
- Schema versionado pelo campo `version`; campos novos só por adição dentro da v1
- O host não retém nada: o estado em memória se perde no restart (sessões, bloqueios, cache e estado dos plugins)
- Retenção do inventário (24 h) e teto de memória (128 MiB) são config do core e aplicados por ele; sem auditoria na v1

---

### Interfaces públicas
| Nome | Tipo | Protocolo | Exposição | SLAs/Limites |
| ---- | ---- | ---------- | --------- | ------------- |
| Servidor gRPC (`17443`, `n3x.plugins.<namespace>.<nome>.v1`) | API | gRPC, TLS 1.3 | Interna (sidecars) | 500 sidecars em 1 vCPU e 512 MB; 1.000 conexões, 16 streams por conexão, mensagens até 4 MiB; teto de 10 s por chamada unária de plugin e de 1 s por mensagem em streams |
| `n3x.plugins.n3x.core.v1.Login` | API | gRPC, TLS 1.3 | Interna, única rota sem sessão | 5 falhas por IP e serviço bloqueiam por 15 min; até 2 bcrypt simultâneos; cache de 1 h |
| Painel (`17444`) | API | HTTPS, TLS 1.3 | Interna (operadores) | p95 < 300 ms com 500 sidecars; polling de 5 s; WCAG 2.1 AA |
| Operação (`17091`: `/metrics`, `/live`, `/ready`) | API | HTTP | Interna (Prometheus e orquestrador) | `/ready` só após todos os plugins iniciarem; 503 no desligamento |
| CLI (`run`, `validate`, `hash`, `credential new`, `encrypt`, `keygen`, `dev-cert`) | SDK | Processo e stdin/stdout | Operador e CI | Segredos só pelo stdin |
| `n3x.yaml` + certificado TLS + chave AES | SDK | Arquivos montados | Orquestrador → container | Lidos só na subida |
| Host → plugin (registradores, identidade, estado, verificador de credencial, sessões) | SDK | Interfaces Go | Plugins compilados no binário | Contrato no HLD do `n3x-sdk` |

**Convenções do servidor gRPC**
- Cada plugin registra serviços só sob `n3x.plugins.<namespace>.<nome>.v<major>`; dois plugins com o mesmo serviço impedem a subida
- Identidade do chamador disponível no `ctx` pelo SDK; nenhum plugin enxerga o token
- Metadados aceitos: `authorization` (token, consumido pelo host) e `x-request-id` (correlação)
- Reflection e health gRPC desligados na porta `17443` (hipótese)

**Códigos de erro do servidor gRPC**
| Situação | Código |
| --- | --- |
| Sem sessão, sessão inválida ou expirada, credencial errada | `UNAUTHENTICATED` |
| Papel sem permissão para o método | `PERMISSION_DENIED` |
| Limite de login, de verificações, de conexões ou de streams | `RESOURCE_EXHAUSTED` |
| Método ou versão desconhecida | `UNIMPLEMENTED` |
| Panic no handler do plugin | `INTERNAL`, com mensagem genérica |
| Plugin `falho` ou controller parando | `UNAVAILABLE` |
| Prazo da chamada estourado | `DEADLINE_EXCEEDED` |
| Campos inválidos ou major do proto incompatível | `INVALID_ARGUMENT` e `FAILED_PRECONDITION`, definidos pelo plugin |

**Painel: mapa de telas**
| Rota | Dono | Papel mínimo |
| --- | --- | --- |
| `/login`, `/logout` | Host | Público / sessão |
| `/` (visão geral com cartões dos plugins e banners de estado) | Host | Leitura |
| `/plugins/n3x/core/sidecars`, `/configs`, `/fluxo` | Plugin core | Leitura |
| `/plugins/n3x/core/usuarios` (lista só leitura: nome e estado) | Plugin core | Root |
| `/plugins/<namespace>/<nome>/...` | Cada plugin | Declarado pela rota (padrão leitura; root para métodos que não são GET) |
| `/assets/...` (htmx, CSS) | Host, embutido | Público |

---

### Considerações de escalabilidade e disponibilidade
Abordagem geral
- Instância única por ambiente, com escala só vertical (ADR-001 do `n3x-sdk`)
- O controller nunca está no caminho da requisição da aplicação: zero erros de tráfego causados por ele
- O estado observado se reconstrói sozinho: todo sidecar refaz login e registro ao reconectar
- Um plugin opcional com problema fica isolado (estado `falho` ou `degradado`), e o resto do controller segue

Técnicas aplicadas
- Limites no gRPC: 1.000 conexões, 16 streams por conexão, mensagens de até 4 MiB, keepalive com política de tempo mínimo para o cliente
- Proteção do bcrypt (ADR-004): até 2 verificações simultâneas, fila curta com prazo, cache de 1 h de credenciais verificadas por serviço e bloqueio por IP e serviço
- `session.hmac_key` configurada na seção do core recomendada, para os tokens sobreviverem ao restart e evitar a rajada de logins (ADR-004 do `n3x-plugin-core`)
- Prazo em toda chamada unária a plugin (`runtime.plugin_call_timeout`) e de 1 s por mensagem processada em streams longos
- Polling de 5 s com respostas parciais em HTML, sem conexão aberta (ADR-008)
- `GOMEMLIMIT` em 80% do limite de memória do container (hipótese)

**Orçamento do controller** (hipótese, confirmado pelo teste de carga com 500 sidecars simulados)
| Parcela | Memória | CPU em regime |
| --- | --- | --- |
| Host e runtime Go | 60 MB | 0,05 vCPU |
| 500 conexões gRPC e streams de heartbeat (50 mensagens/s) | 100 MB | 0,15 vCPU |
| Plugin core e dados em memória | 150 MB | 0,15 vCPU |
| Folga (picos de login, painel, GC) | 200 MB | 0,65 vCPU |
| **Total (meta do PRD)** | **512 MB** | **1 vCPU** |

- Pior caso de login após restart sem `session.hmac_key` no core: um bcrypt por serviço (cerca de 250 ms cada), ou cerca de 12 s de CPU para 50 serviços, contra cerca de 2 min sem o cache

Meta de disponibilidade
- Controller com 99,5% de disponibilidade mensal, apoiado no restart do orquestrador
- Sidecar de volta ao painel em até 30 s após o controller voltar: cerca de 10 s com `session.hmac_key` configurada no core; sem ela, vale até cerca de 200 serviços, limitado pelo bcrypt por serviço (ADR-004 do `n3x-plugin-core`)
- Painel em p95 < 300 ms com 500 sidecars conectados
- Com o core `degradado` (teto de memória estourado, ADR-009 do `n3x-plugin-core`), o painel continua em leitura com banner e só o root entra

---

### Segurança
Autenticação
- Sidecar: login em `n3x.plugins.n3x.core.v1.Login` com a credencial do serviço; o host confere o bcrypt (custo ≥ 12) contra o `secret_hash` único (ADR-005) e o core emite o token HMAC-SHA256 por instância (ADR-005 do `n3x-plugin-core`); nas chamadas seguintes, o token é conferido pelo autenticador do core (ADR-003 e ADR-004 do `n3x-sdk`)
- Cache de credencial verificada: guarda só um HMAC do segredo, com chave aleatória do processo, por 1 h; falhas nunca entram no cache (ADR-004)
- Painel: root conferido pelo bcrypt do arquivo; usuários de leitura conferidos pelo diretório do core; sessão só em memória, com ID aleatório de 32 bytes (ADR-006)
- Rotação da credencial de serviço: troca do hash, restart do controller e redeploy do serviço; os sidecars somem do painel durante a troca, sem afetar o tráfego
- Troca da senha do root: novo hash no arquivo e restart
- Usuários de leitura: criar, desativar, remover ou trocar senha é editar a seção do core no `n3x.yaml` e reiniciar; não há troca de senha pelo painel na v1 (ADR-003 do `n3x-plugin-core`)

Autorização
- Papéis `root`, `leitura` e `sidecar`, aplicados pelo host antes de qualquer handler de plugin
- gRPC: só o papel `sidecar` na v1 (ADR-007)
- Painel: papel mínimo declarado por rota; a lista de usuários exige root; 403 também no servidor
- Capacidades privilegiadas só para o core: rota pública de login, autenticador de sessão, diretório de usuários e uso do verificador de credencial (ADR-004 do `n3x-sdk` e ADR-003); o gravador de auditoria fica reservado para a v2
- Namespaces: serviços só em `n3x.plugins.<namespace>.<nome>.v<major>` e rotas só em `/plugins/<namespace>/<nome>/`, conferidos na subida

Proteção de dados
- Cookie `n3x_session` com `HttpOnly`, `Secure` e `SameSite=Strict`; expira após 30 min sem uso ou 12 h no total
- CSRF por token de sessão em todo método que não é GET
- CSP `default-src 'self'`, sem script inline nem JavaScript externo; htmx e CSS servidos pelo binário
- Mensagem de login genérica; bloqueio de 5 tentativas por 15 min por usuário e IP
- TLS 1.3 em `17443` e `17444`, com certificado em arquivo; `dev-cert` só para desenvolvimento
- Nada de segredo, token, cookie, hash ou corpo de requisição em log ou mensagem de erro
- LGPD: o host não guarda dados pessoais; IP e usuário dos logins do painel aparecem só no log JSON operacional, com a retenção do coletor de logs; a trilha formal com retenção fica para a v2 (ADR-002 do `n3x-plugin-core`)
- Endurecimento: usuário não root, sistema de arquivos raiz somente leitura, `drop: ALL`, binário estático sem cgo, imagem fixada por digest, `govulncheck` e `gosec` na CI

Gestão de segredos
- bcrypt para o que só é conferido: root e credenciais de serviço (usuários de leitura ficam com o core)
- `ENC[v1:...]` com AES-256-GCM para segredos das seções de plugin (ex.: `session.hmac_key` do core), no mesmo formato do sidecar
- Chave AES só em arquivo montado, nunca em variável de ambiente
- Ferramentas: `hash`, `credential new`, `encrypt` e `keygen`, com segredos lidos do stdin

---

### Observabilidade
Logs
- JSON em stderr com `ts`, `level`, `component` (`host`, `grpc`, `panel` ou `plugin`), `plugin_id`, `caller` (usuário ou serviço+instância) e `request_id`
- Subida: plugins na ordem de início, estado de cada um e valores efetivos de limites e `GOMEMLIMIT`
- Login, logout e bloqueio do painel com usuário, IP e resultado (paliativo da auditoria, que fica para a v2)
- Stack trace de panic amostrado em no máximo 1 por segundo por plugin
- Nunca registra segredos, tokens, cookies, hashes ou corpos de requisição

Métricas (`17091`, prefixo `n3x_controller_`)
| Métrica | Tipo | Rótulos |
| --- | --- | --- |
| `grpc_calls_total` | counter | `method`, `code` |
| `grpc_duration_seconds` | histogram | `method` |
| `grpc_connections` / `grpc_streams` | gauge | |
| `logins_total` | counter | `result`, `service` |
| `login_blocked_total` | counter | `kind` (`sidecar` ou `panel`) |
| `bcrypt_inflight` | gauge | |
| `credential_cache_hits_total` | counter | |
| `panel_requests_total` | counter | `route`, `status` |
| `panel_duration_seconds` | histogram | `route` |
| `panel_sessions` | gauge | |
| `plugin_state` | gauge 0/1 | `plugin`, `state` |
| `plugin_panics_total` | counter | `plugin`, `kind` (`grpc` ou `panel`) |
| `build_info` | gauge = 1 | `version`, `plugins_hash` |

- `route` usa o padrão da rota (ex.: `/plugins/n3x/core/sidecars/:id`), nunca o path real
- Métricas de negócio (sidecars conectados, divergências) são expostas pelo plugin core com o prefixo `n3x_n3x_core_` (ADR-010 do `n3x-plugin-core`)

Tracing
- Sem tracing distribuído na v1 (ADR-009 do `n3x-sdk`)
- Correlação pelo `request_id` recebido do sidecar nos metadados gRPC e gerado no painel

Dashboards e alertas
- Alertas sugeridos:
  - `histogram_quantile(0.95, panel_duration_seconds) > 0.3`
  - `plugin_state{state!="ok"} == 1`
  - `rate(plugin_panics_total[5m]) > 0`
  - `rate(login_blocked_total[5m]) > 0`
  - `/ready` fora por mais de 1 minuto
- Painel de referência (hipótese, em `n3x-sdk/deploy/`): chamadas gRPC por método e código, latência do painel, logins, estado dos plugins e memória em relação ao limite

---

### Riscos arquiteturais e mitigação
#### O plugin core concentra login, usuários e inventário
- **Probabilidade:** média
- **Impacto:** um bug no core deixa o painel sem login de leitura e sem dados; o root continua entrando e o tráfego não é afetado
- **Mitigação:**
  - TDD e testes de conformidade dedicados ao core
  - Core obrigatório: o controller não sobe sem ele
  - Root independente do core, conferido pelo arquivo
- **Plano de contingência:** rollback da imagem do controller para a versão anterior

#### Core degradado por teto de memória
- **Probabilidade:** baixa
- **Impacto:** usuários de leitura não entram e novas versões de config são recusadas; heartbeats e estados seguem (ADR-009 do `n3x-plugin-core`)
- **Mitigação:**
  - O core informa `degradado` e o banner aparece
  - Descarte de histórico, eventos antigos e configs sem instância antes de degradar
  - Métrica `n3x_n3x_core_degraded` com alerta
- **Plano de contingência:** reiniciar o controller ou subir `memory_limit`; o inventário se reconstrói pelos sidecars

#### Rajada de logins após restart do controller
- **Probabilidade:** média
- **Impacto:** CPU saturada por bcrypt, painel lento e sidecars demorando a voltar
- **Mitigação:**
  - Cache de credencial verificada por serviço (ADR-004)
  - Até 2 bcrypt simultâneos e fila com prazo
  - Jitter no novo login do sidecar
  - `session.hmac_key` configurada no core
- **Plano de contingência:** escala vertical temporária do controller

#### Panic ou travamento em tela ou serviço de plugin
- **Probabilidade:** média
- **Impacto:** erro 500 ou `INTERNAL` naquela chamada; um erro fatal do Go derruba o processo
- **Mitigação:**
  - Recover nos interceptors e middlewares
  - Prazo em toda chamada de plugin
  - Plugin opcional que falha na subida fica desligado
- **Plano de contingência:** o orquestrador reinicia o controller; o plugin sai da imagem e o controller é redeployado

#### Plugin acessa o que não deve
- **Probabilidade:** baixa
- **Impacto:** um plugin lê dados ou tokens de outro, ou expõe uma rota sem autenticação
- **Mitigação:**
  - Checagem de namespace e rotas na subida
  - Capacidades privilegiadas só para o core
  - Autorização no host antes de qualquer handler
  - Análise estática no build (`unsafe`, `reflect`, pacotes internos do host)
- **Plano de contingência:** remover o plugin da imagem e redeployar

#### Meta de 500 sidecars em 512 MB não fecha
- **Probabilidade:** média
- **Impacto:** OOM ou painel acima de 300 ms
- **Mitigação:**
  - Teste noturno com 500 sidecars simulados (`test/load/`)
  - `GOMEMLIMIT` e limites de conexões e streams
  - Polling com respostas parciais
- **Plano de contingência:** rever o orçamento com base na medição ou recomendar 1 GB

#### Restart do controller desloga todos do painel
- **Probabilidade:** alta
- **Impacto:** baixo; os operadores fazem login de novo
- **Mitigação:**
  - Risco aceito em troca do corte imediato de sessões (ADR-006)
  - Volta à mesma página após o login
- **Plano de contingência:** se incomodar, migrar para cookie assinado numa versão futura

---

### ADRs e próximos passos
ADRs associados
- [ADR-001: Host como biblioteca, com o binário sempre gerado pelo `n3x-plugin build`](adr/ADR-001-host-como-biblioteca.md)
- [ADR-002: Controller sem armazenamento, cada plugin traz o próprio banco](adr/ADR-002-controller-sem-armazenamento.md)
- [ADR-003: Usuários de leitura e auditoria no plugin core](adr/ADR-003-usuarios-e-auditoria-no-core.md) (substituído parcialmente em 2026-10-20)
- [ADR-004: Verificador de credencial no host, com cache de 1 h](adr/ADR-004-verificador-de-credencial-no-host.md)
- [ADR-005: Um único hash por serviço](adr/ADR-005-um-hash-por-servico.md)
- [ADR-006: Sessão do painel só em memória](adr/ADR-006-sessao-do-painel-em-memoria.md)
- [ADR-007: Servidor gRPC só para sidecars na v1](adr/ADR-007-grpc-so-para-sidecars.md)
- [ADR-008: Painel com polling de 5 s e visão geral por cartões](adr/ADR-008-polling-e-cartoes.md)
- [ADR-009: Subcomandos do binário do controller](adr/ADR-009-subcomandos-do-controller.md)
- Do `n3x-plugin-core` (afetam o host): ADR-001 (armazenamento em memória), ADR-002 (auditoria na v2), ADR-003 (usuários pelo yaml), ADR-004 (chave HMAC do core), ADR-005 (token de sessão) e ADR-009 (teto de memória)
- Herdados do `n3x-sdk`: ADR-001 (instância única por ambiente), ADR-003 (login e sessão HMAC), ADR-004 (capacidades do plugin core), ADR-005 (memória isolada), ADR-006 (portas 17xxx), ADR-008 (exemplos de deploy) e ADR-009 (sem tracing). O ADR-002 do `n3x-sdk` (armazenamento efêmero ou Postgres) é substituído pelo ADR-002 deste repositório

Decisões pendentes
- Contrato das capacidades novas do core (diretório de usuários, uso do verificador de credencial) e da API de cartões e de estado: HLD do `n3x-sdk`; gravador de auditoria e aviso de usuário desativado sem uso na v1
- Resolvida: banco do plugin core. Na v1 tudo fica em memória, com teto e degradação (ADR-001 e ADR-009 do `n3x-plugin-core`); banco real na v2
- Resolvida: emissão do token HMAC pelo core (ADR-004 e ADR-005 do `n3x-plugin-core`)
- Resolvida: o pacote `secret` foi para o `n3x-sdk` (ADR-014 do `n3x-sdk`)
- Imagem base do controller (distroless static nonroot é hipótese)
- Ajuste do orçamento de memória e de `GOMEMLIMIT` após o primeiro teste de carga

Próximos passos
- Implementar `host/config` com o schema v1, testes de tabela e fuzzing, base do `validate`
- Implementar `host/plugins` e `host/grpcsrv` com TDD, plugins falsos e servidor em memória (`bufconn`)
- Implementar `host/credential` e `host/session` com testes de bloqueio, cache, CSRF e permissões
- Montar o layout do painel com axe na CI
- Criar o `cmd/n3x-controller-dev`, o `dev/n3x.yaml` e o `go.work` de desenvolvimento
- Escrever o simulador de 500 sidecars em `test/load/` e medir o orçamento
- Escrever o FDD/LLD do host de plugins e do servidor gRPC
