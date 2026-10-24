# DRAFT_CONSTITUITION: n3x-controller

Insumo para gerar a constituição do repositório com o Spec Kit (`/speckit-constitution`). Não é a constituição.
Fontes: PRD 1.6 (cópia do controller), HLD 1.2, ADR-001 a ADR-009, FDD-CTL-01 a FDD-CTL-07, `specs/README.md` e decisões do mantenedor em 2026-10-23. Para o deploy: FDD-SDK-15 (`n3x-sdk`) e FDD-SID-11 (`n3x-sidecar`).

## 1. Identidade

- O N3X é um middleware de rede extensível: uma camada sobre o Envoy em modelo sidecar. Cada aplicação ganha um sidecar como única porta de entrada, e os sidecars se conectam a um controller central que só observa. Comportamentos de rede viram plugins Go com até duas metades, a de proxy e a de controller.
- O `n3x-controller` é esse controller: um host de plugins sem armazenamento. Hospeda as metades de controller e oferece servidor gRPC único (`17443`, TLS 1.3) aos sidecars, painel HTTPS (`17444`) aos operadores e operação (`17091`: `/metrics`, `/live`, `/ready`).
- Só observa: nunca altera a config de um sidecar e fica fora do caminho do tráfego. Controller fora do ar não causa erro de requisição.
- Entrega o módulo Go `host/` e a imagem base `ghcr.io/giustech/n3x/controller:<semver>`, sem binário (ADR-001). O binário sai do `n3x-plugin build`, com um `main` que chama `host.Run(host.Core(...), host.Plugin(...)...)`.
- Tudo é plugin: o host só serve o que as metades compiladas registram. O plugin core (`n3x.core`) é obrigatório e sem ele o controller não sobe.
- Problema que resolve: um ponto único para ver sidecars, configs efetivas e divergências, com autenticação forte, isolamento entre plugins e contenção de falhas, sem banco e dentro de 1 vCPU e 512 MiB para 500 sidecars.
- Requisitos do PRD: FR-005, FR-008, FR-009 e FR-011 inteiros; FR-006, FR-007 e FR-016 no papel de host (dados e lógica no core); FR-015 na imagem e na compatibilidade do `host/`. FR-010 (auditoria) fica para a v2.

## 2. Quem usa e onde

- Operador (time de plataforma/SRE, persona principal): escreve o `n3x.yaml`, gera hashes, credenciais, chave e `ENC[...]` pelos subcomandos, roda `validate` na CI e lê painel, métricas e logs.
- Root (único, vindo do arquivo) e usuários de leitura (da seção do core): usam o painel. Só o root vê a lista de usuários.
- Sidecars: fazem login e chamam o gRPC com o papel `sidecar`, pela conexão compartilhada do `.so`.
- Autor de plugin: recebe do host o `Env`, os registradores (gRPC, páginas, ações, cartões), memória isolada e canal de estado, e recusas claras na subida. Só o core recebe o `CoreEnv` com capacidades privilegiadas.
- Também: builder (gera o `main` e embute o manifesto), orquestrador (monta arquivos, envia sinais, consulta probes) e Prometheus.
- Onde: qualquer ambiente com containers (Kubernetes, ECS, docker compose), em instância única por ambiente e escala só vertical. Topologias de referência em `n3x-sdk/deploy/`. Com Istio, o controller fica fora da malha ou com a `17443` fora da captura (SID-11).

## 3. Escopo e limites

- Dono de: schema v1 do `n3x.yaml` do controller; subida, desligamento e CLI; códigos `N3X-C001` a `N3X-C008`; servidor gRPC (convenção `n3x.plugins.<namespace>.<nome>.v<major>`, cadeia, limites e códigos); host de plugins (ordem, obrigatório e opcional, contenção, estado `ok`, `degraded` e `failed`); verificador de credencial; sessão, login e CSRF do painel; layout, slots e visão geral; métricas `n3x_controller_*`; imagem base; simulador `test/load`.
- Consome, sem ser dono: contratos do `n3x-sdk` (`ctrl`, `controller/ui`, `hostrules`, `memory`, `telemetry`, `plugin`, `secret`, `artifact`, `yamlconv`, `sdktest/hostconformance`); o `n3x-plugin-core` (rota de login, token HMAC, autenticador, diretório de usuários, inventário e telas); o manifesto ELF e o `main` gerados pelo `n3x-builder`.
- Sem armazenamento (ADR-002): nem banco, nem migrations. Sessões, bloqueios, cache e estado de plugin vivem em memória e somem no restart. Plugin que guarda dados traz o próprio banco.
- Config só por arquivo, lida só na subida. Nenhum campo vem de variável de ambiente.
- Fora da v1: auditoria, gestão de usuários pelo painel, papéis por método no gRPC, mTLS de sidecar, SSE, várias réplicas, recarga de config ou certificado sem restart, reinício de plugin `failed`, reflection e health gRPC na `17443`, tracing.

## 4. Stack técnica

- Go 1.26 ou superior, sem cgo, binário estático. Produção em `linux/amd64` e `linux/arm64`; desenvolvimento em Linux, macOS ou WSL2.
- grpc-go 1.67 ou superior e protobuf; Echo v5 (5.4.0), templ e htmx 2.0.4 embutido; `golang.org/x/crypto/bcrypt`; `log/slog` em JSON pelo handler do SDK; `prometheus/client_golang`; YAML pelo `yamlconv` do SDK.
- CLI com o pacote `flag` da stdlib, no binário e no `n3x-load`, sem `cobra`. `testify` em todos os testes, junto da stdlib.
- Testes também usam `bufconn`, `httptest`, `goleak`, fuzzing nativo, golden files, `secrettest` e `hostconformance`. Playwright, Chromium e axe (Node 22) só na CI do painel.
- Imagem `gcr.io/distroless/static-debian12` fixada por digest, `USER 17000:17000`, sem shell (por isso o subcomando `probe`), índice OCI amd64 e arm64 no GHCR.
- CI no GitHub Actions: `ci.yml` em todo PR e `load.yml` (`smoke`, `nightly`, `prerelease`) na máquina dedicada `n3x-perf`.

## 5. Arquitetura interna

- Só o pacote `host` é API estável, com o semver do módulo `host/`. Componentes em `host/internal/`: `config`, `preflight`, `ops`, `boot`, `cli`, `credential`, `plugins` (e `pluginstest`), `grpcsrv`, `clientip`, `session` e `panel`.
- Também: `cmd/n3x-controller-dev` (tag `n3x_dev`), `dev/`, `images/`, `ci/a11y/` e `test/load/` (módulo aninhado com a CLI `n3x-load`).
- Montagem fixa: `credential` → `session.Store` → `plugins` → `Start` → `session.NewLogin` → `grpcsrv` → `panel`. Construtores devolvem erro (saída 8). Só o `host.Run` trata sinais e decide a saída.
- Arquitetura hexagonal formal: regras no núcleo, portas como interfaces Go e adaptadores separados para relógio, aleatoriedade, bcrypt, listeners, sistema de arquivos, cgroup, leitura do manifesto e Docker (no `test/load`). Os campos de função das `Options` das FDDs viram portas.
- Regras de dependência: `grpcsrv` e `plugins` sem Echo nem templ; `clientip` só com stdlib; `pluginstest`, `credentialtest` e `NewWithOptions` só em teste (a CI confere); o core importa só o SDK, nunca o `host/`.

## 6. Princípios candidatos

1. Host sem armazenamento: tudo o que é durável pertence a um plugin (ADR-002).
2. Tudo é plugin; o core é obrigatório e o único com capacidades privilegiadas.
3. Fora do caminho crítico: o estado observado se reconstrói pelos sidecars depois de um restart.
4. Autenticação e autorização no host, antes de qualquer handler de plugin; namespaces gRPC e de rotas conferidos na subida.
5. Contenção de falhas: recover, prazo em toda chamada a plugin e estado `failed` com 5 panics em 60 s ou mais de 100 chamadas presas; opcional falho não derruba o processo.
6. Falha rápida e clara na subida: campo desconhecido é erro, manifesto e versão do `host/` conferidos, mensagem com caminho, código e como resolver.
7. Segredo nunca vaza: nada de segredo, token, hash, cookie ou corpo em log, métrica, erro ou HTML; segredos só por stdin ou arquivo montado.
8. Orçamento como contrato: 500 sidecars em 1 vCPU e 512 MiB e painel em p95 abaixo de 300 ms, medidos toda noite.
9. Evolução só por adição: schema v1, códigos, métricas e protos.
10. Painel acessível e sem JavaScript externo: WCAG 2.1 AA, CSP `'self'` e nenhum plugin escreve HTML.

## 7. Contratos públicos

- CLI: `run` (padrão), `validate`, `hash`, `credential new <serviço>`, `encrypt --key --plugin <id>`, `keygen`, `dev-cert`, `probe [live|ready]`, `version` e `help`. Segredos só pelo stdin, e terminal é recusado.
- Saídas: 0 ok; 1 uso; 2 config; 3 segredo ou chave; 4 manifesto, imagem ou registro; 5 plugin na subida; 6 porta; 7 desligamento forçado; 8 interno.
- `n3x.yaml` em `/etc/n3x/controller.yaml` ou `--config`: `version`, `listen`, `tls`, `keys`, `root`, `services.<nome>.secret_hash` (um por serviço), `grpc`, `panel`, `runtime` (com `plugin_limits`), `resources` e `plugins.<id>`. bcrypt de custo 12 a 16; `ENC[...]` em AES-256-GCM com AAD `n3x:v1:controller:<id>`.
- Montagens: o `controller.yaml` vem de um `Secret` (tem hashes bcrypt); `aes.key`, `tls.crt` e `tls.key` ficam em `/run/n3x/` (SDK-15).
- `17443`: gRPC, TLS 1.3 e ALPN `h2`, só o papel `sidecar`; `n3x.plugins.n3x.core.v1.Login` é a única rota pública.
- `17444`: HTTPS TLS 1.3; cookie `n3x_session` (30 min ocioso, 12 h no total); CSRF em todo método que não é GET.
- `17091`: HTTP interno; `/ready` responde `starting`, `ready`, `plugin_failed` ou `stopping`.
- Códigos gRPC do host: `UNAUTHENTICATED`, `PERMISSION_DENIED`, `RESOURCE_EXHAUSTED`, `UNIMPLEMENTED`, `FAILED_PRECONDITION` (major do core), `INTERNAL`, `UNAVAILABLE` e `DEADLINE_EXCEEDED`.
- Logs JSON em stderr (`ts`, `level`, `component`, `msg`, `plugin_id`, `caller`, `request_id`). SIGTERM e SIGINT desligam, um segundo sinal sai com 7 e SIGHUP é ignorado.

## 8. Requisitos não funcionais

- 500 sidecars em 1 vCPU e 512 MiB; painel em p95 abaixo de 300 ms; 99,5% de disponibilidade mensal; sidecars de volta ao painel em até 30 s após um restart: cerca de 10 s com `session.hmac_key` e, sem ela, até cerca de 100 serviços por vCPU.
- Orçamento, hipótese até a carga medir: host 60 MiB e 0,05 vCPU; conexões 100 MiB e 0,15; core 150 MiB e 0,15; folga 200 MiB e 0,65. `GOMEMLIMIT` em 80% do limite.
- Login de sidecar: até 2 bcrypt simultâneos, fila de 16 e espera de 5 s; cache de 1 h por serviço; bloqueio com 5 falhas por IP ou 5 IPs distintos por serviço, numa janela de 15 min. Até 5 bcrypt no processo, somando o painel.
- gRPC: 1.000 conexões, 16 streams por conexão, mensagens de 4 MiB, 10 s por unária e 1 s por mensagem em stream.
- Prazos: subida de plugin 30 s, desligamento 20 s e `Stop` de 5 s por plugin. O orquestrador precisa de grace de pelo menos `shutdown_timeout + 2 s + 5 s × N + 1 s`.

## 9. Qualidade, testes e CI

- TDD com testes de tabela por regra, por código e por linha de contrato; `go test -race`; fuzzing nos parsers; golden de `/metrics` e do layout; `goleak`; canário de segredo com `secrettest.Leaks`.
- Conformidade com o `hostconformance` do SDK (79 casos) contra o harness `pluginstest`.
- Painel: axe (WCAG 2.1 AA) com Playwright nos temas claro e escuro, zero violações de CSP e navegação por teclado.
- Portões: `go vet`, `gosec`, `govulncheck`, `templ generate` sem diferença, `go list -deps` sem pacotes proibidos e benchmarks de alocação.
- Cobertura mínima de linhas: 85% em geral e 90% nos pacotes críticos (`config`, `preflight`, `credential`, `plugins`, `grpcsrv`, `clientip`, `session` e `panel`, sem os `*_templ.go`). O `test/load` fica fora da conta.
- Carga com o `n3x-load`: `smoke` em PR que mexe nos pacotes quentes, `nightly` e `prerelease`, que bloqueia a tag. O marco M5 pede 3 noites verdes seguidas e uma `prerelease` verde no mesmo commit.

## 10. Governança e fluxo

- Licença Apache 2.0. Mantenedor único: Guilherme Giusti.
- Fluxo PRD → HLD → FDD → Spec Kit, na ordem de `specs/README.md` (01 a 07). A FDD é a fonte de verdade da spec.
- Precedência: quando uma FDD diverge do HLD, de um ADR ou do PRD, vale a FDD, e o documento de cima é atualizado.
- Uma branch `NNN-slug` do Spec Kit por feature e PR obrigatório para a `main`, com a CI como portão. Mensagens em Conventional Commits. Todo merge é humano.
- Idioma: documentação e mensagens ao operador em pt-BR; identificadores, campos, métricas e códigos em inglês.
- Pré-requisitos externos: SDK com `artifact` v0.2.0 ou superior; core mínimo `v0.1.0` (CORE-01) antes da spec 03; diretório de usuários do core (CORE-04) para as specs 05 e 06; FDDs CORE-01 a CORE-09 para a spec 07.

## 11. Documentos a atualizar antes do Spec Kit

- PRD: consolidar a 1.7, fundindo as duas 1.6 (controller e SDK) e a 1.5 do builder, e replicar nos 5 repositórios. No controller:
  - FR-005: `FAILED_PRECONDITION` para major do core; métricas sem rótulo de identidade (ela fica só no log).
  - FR-008: plugin `failed` sai do menu e responde 503. FR-009: bloqueio em janela fixa de 15 min.
  - Disponibilidade e critérios de aceite: sem `session.hmac_key`, cerca de 100 serviços por vCPU.
  - Compliance: usuário e IP dos logins do painel e do sidecar no log operacional, com a retenção do coletor.
  - Também: papel `plugin`, MB lido como MiB, soak de 60 min do controller na `prerelease`.
- Remover `docs/prd/PRD-N3X.json`, `docs/HLD.json` e `docs/fdd/*.json`, e os links no `specs/README.md`.
- `HLD.md` (1.3):
  - árvore em `host/internal/`; `host.Run(host.Core(...), host.Plugin(...)...)`; etapa 0 com o manifesto antes da config;
  - registro depois do `Start`; ordem da cadeia gRPC (limites no transporte, log fora do recover) e recover só nos interceptors;
  - `FAILED_PRECONDITION`, metadados aceitos, `grpc_duration_seconds` só de unárias, rótulo `method` e rota `{nome}`;
  - `component` também `credential`; `go run -tags n3x_dev`; contador de serviço por IPs distintos; MiB;
  - montagens da SDK-15, controller fora da malha Istio (SID-11), cerca de 100 serviços por vCPU, `pprof` e `runtime/metrics`.
- ADR-001: assinatura do `host.Run` e imagem `ghcr.io/giustech/n3x/controller`. ADR-004: 2 bcrypt de sidecar, fila de 16, espera de 5 s, IPs distintos, 5 no processo e cerca de 100 serviços por vCPU. ADR-006: o corte das sessões de leitura pelo estado do core vale na v1.
- Todas as FDDs: versão 1.1, citação do PRD 1.7, portas como interfaces (hexagonal), `testify`, IDs sem repetição e sem notas "HLD a atualizar" vencidas.
- CTL-01: `plugin_failed` em `ops.State`; sem `Report.Runtime`; avisos de runtime fora do `validate`; pacote `cli` listado; `/debug/pprof` só com `n3x_dev` e coletor `runtime/metrics` na `17091`; exemplos com `/run/n3x/` e `Secret`.
- CTL-02: teto de 2 bcrypt é do login de sidecar, não do processo.
- CTL-04: com o core `failed`, a regra da major vem antes (I-09); `clientip` com 100% de linhas.
- CTL-05: sem contador por IP; teto de memória coerente com 1.024 sessões.
- CTL-06: método inválido sem sessão responde 405; 404 de asset em `text/plain`; cobertura de 90%.
- CTL-07: remover `setup --cost` abaixo de 12; `scale-nokey` vira meta dura de 100 serviços em 1 vCPU; `--cap-drop ALL` no container de carga.
- `specs/README.md`: pré-requisitos externos de cada spec (a 07 depende das FDDs do core).

## 12. Pendências conhecidas

- Metas de desempenho e o orçamento seguem como hipótese até a primeira carga: keepalive, fila de bcrypt, `GOMEMLIMIT`, `MaxOverviewCards` e custos por chamada.
- APIs experimentais do grpc-go (`InTapHandle`, `stats.Handler`), com contingência nos interceptors (P-04 da CTL-04).
- Pedidos a FDDs do SDK: regra `ambiguous_route` (SDK-08), classes do layout (SDK-10), `PanelLogin` no `hostconformance` e critério de `failed` em `hostrules` (SDK-12).
- Core `v0.1.0` sem `CheckUser` real: os casos de login do painel ficam pulados até CORE-03 e CORE-04.
- Revisões de texto sem decisão: ordem de bloqueio e consulta do serviço na CTL-02; tabela de 30 vetores de path e 413/415 no axe na CTL-06; `smoke` com `--binary`, duração do `nightly` e FDD 44 nas dependências da CTL-07.
