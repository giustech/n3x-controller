<!--
Sync Impact Report
- Versão: (modelo sem preenchimento) → 1.0.0
- Princípios: criados I a X (primeira ratificação; nenhum renomeado).
- Seções adicionadas: Restrições Técnicas e de Contrato; Fluxo de Desenvolvimento e Portões de Qualidade; Governança.
- Seções removidas: nenhuma.
- Templates:
  - ✅ .specify/templates/plan-template.md (Constitution Check com os portões I a X)
  - ✅ .specify/templates/tasks-template.md (testes obrigatórios, não opcionais)
  - ✅ .specify/templates/spec-template.md (sem mudança necessária)
  - ✅ CLAUDE.md (já exige PT-BR; coerente com a seção de fluxo)
- TODOs adiadas: nenhuma.
-->

# Constituição do n3x-controller

O `n3x-controller` é o host de plugins sem armazenamento do N3X: oferece um servidor gRPC único
(`17443`) aos sidecars, um painel HTTPS (`17444`) aos operadores e a porta de operação (`17091`).
Ele só observa. Esta constituição vale para toda spec, plano, tarefa e código do repositório.

## Princípios Fundamentais

### I. Host sem armazenamento
O host MUST NOT ter banco, migrations ou qualquer persistência própria. Sessões, bloqueios, cache e
estado de plugin vivem em memória e somem no restart. Todo dado durável pertence a um plugin, que
traz o próprio banco. Config é lida só do arquivo `n3x.yaml`, só na subida; nenhum campo vem de
variável de ambiente. Motivo: ADR-002; o controller precisa ser descartável e reiniciável.

### II. Tudo é plugin; o core é obrigatório
O host só serve o que as metades de controller compiladas registram. O plugin `n3x.core` é
obrigatório e, sem ele, o controller MUST NOT subir. Só o core recebe o `CoreEnv` com capacidades
privilegiadas; o core importa só o SDK, nunca o `host/`. Funcionalidade nova entra como plugin ou
como contrato do SDK, não como atalho dentro do host.

### III. Fora do caminho crítico
O controller só observa: MUST NOT alterar a config de um sidecar nem ficar no caminho do tráfego.
Controller fora do ar MUST NOT causar erro de requisição. O estado observado precisa ser
reconstruído pelos sidecars depois de um restart (alvo: de volta em até 30 s).

### IV. Autenticação e autorização no host
Autenticação e autorização MUST ocorrer no host, antes de qualquer handler de plugin. Os namespaces
gRPC (`n3x.plugins.<namespace>.<nome>.v<major>`) e de rotas são conferidos na subida. A única rota
pública é `n3x.plugins.n3x.core.v1.Login`. TLS 1.3 é obrigatório nas portas `17443` e `17444`. CSRF
vale para todo método que não é GET.

### V. Contenção de falhas
Todo plugin roda com `recover` e prazo em toda chamada. Um plugin vira `failed` com 5 panics em
60 s ou mais de 100 chamadas presas. Plugin opcional em `failed` MUST NOT derrubar o processo;
plugin obrigatório em `failed` marca `/ready` como `plugin_failed`. O `recover` fica só nos
interceptors, e só o `host.Run` trata sinais e decide a saída.

### VI. Falha rápida e clara na subida
Campo desconhecido no `n3x.yaml` é erro. Manifesto e versão do `host/` são conferidos antes da
config. Toda recusa MUST trazer caminho, código (`N3X-C001` a `N3X-C008`) e como resolver. Os
códigos de saída do processo (0 a 8) são contrato.

### VII. Segredo nunca vaza (NON-NEGOTIABLE)
Segredo, token, hash, cookie ou corpo de requisição MUST NOT aparecer em log, métrica, erro ou HTML.
Segredos entram só por stdin ou arquivo montado, e a CLI recusa terminal. Métricas não levam rótulo
de identidade; a identidade fica só no log. Todo teste de fluxo com segredo MUST usar o canário
`secrettest.Leaks`.

### VIII. Orçamento como contrato
500 sidecars em 1 vCPU e 512 MiB, painel com p95 abaixo de 300 ms e 99,5% de disponibilidade
mensal. O orçamento é medido toda noite pelo `n3x-load`. Valores marcados como hipótese
(keepalive, fila de bcrypt, `GOMEMLIMIT`, custos por chamada) só viram fato depois da medição.
Mudança que piora o orçamento MUST ser justificada no plano.

### IX. Evolução só por adição
Schema v1 do `n3x.yaml`, códigos de erro e de saída, métricas `n3x_controller_*`, protos e o pacote
`host` MUST evoluir apenas por adição dentro da v1. Remover ou redefinir exige nova major e emenda
desta constituição se afetar um princípio. Só o pacote `host` é API estável; o resto fica em
`host/internal/`.

### X. Painel acessível e sem JavaScript externo
O painel segue WCAG 2.1 AA, usa CSP `'self'` e embute o htmx. Nenhum plugin escreve HTML: eles
registram páginas, ações e cartões pelos registradores do host, e o host renderiza com templ.

## Restrições Técnicas e de Contrato

- **Stack**: Go 1.26 ou superior, sem cgo, binário estático (`linux/amd64` e `linux/arm64`);
  grpc-go 1.67+, Echo v5, templ, `log/slog` em JSON, `prometheus/client_golang`, YAML pelo
  `yamlconv` do SDK. A CLI usa o pacote `flag` da stdlib, sem `cobra`.
- **Entrega**: módulo Go `host/` e imagem base `ghcr.io/giustech/n3x/controller:<semver>`, sem
  binário (ADR-001). Imagem `distroless/static-debian12` fixada por digest, `USER 17000:17000`.
- **Arquitetura**: hexagonal. Regras no núcleo, portas como interfaces Go e adaptadores separados
  (relógio, aleatoriedade, bcrypt, listeners, sistema de arquivos, cgroup). Construtores devolvem
  erro. `grpcsrv` e `plugins` não importam Echo nem templ; `clientip` usa só a stdlib;
  `pluginstest`, `credentialtest` e `NewWithOptions` só existem em teste.
- **Fora da v1**: auditoria, gestão de usuários pelo painel, papéis por método no gRPC, mTLS de
  sidecar, SSE, várias réplicas, recarga de config sem restart, reflection e tracing. Propor
  qualquer um deles MUST passar antes pela emenda do escopo.
- **Idioma**: documentação, specs, planos, tarefas e mensagens ao operador em PT-BR; identificadores,
  campos, métricas, códigos, branches e nomes de arquivo em inglês.

## Fluxo de Desenvolvimento e Portões de Qualidade

- **Fluxo**: PRD → HLD → FDD → Spec Kit, na ordem de `specs/README.md` (01 a 07). A FDD é a fonte
  de verdade da spec; quando diverge do HLD, de um ADR ou do PRD, vale a FDD e o documento de cima
  é atualizado.
- **Testes primeiro**: TDD com testes de tabela por regra, por código e por linha de contrato. Os
  testes MUST ser escritos e falhar antes da implementação. Usa-se `testify`, `go test -race`,
  `goleak`, fuzzing nos parsers e golden files (`/metrics`, layout). O harness `pluginstest` MUST
  passar no `hostconformance` do SDK.
- **Cobertura mínima de linhas**: 85% em geral e 90% nos pacotes críticos (`config`, `preflight`,
  `credential`, `plugins`, `grpcsrv`, `clientip`, `session`, `panel`, sem `*_templ.go`).
- **Portões da CI**: `go vet`, `gosec`, `govulncheck`, `templ generate` sem diferença,
  `go list -deps` sem pacotes proibidos, benchmarks de alocação e, no painel, axe com Playwright
  nos temas claro e escuro sem violações de CSP.
- **Carga**: `smoke` em PR que mexe em pacote quente, `nightly` todas as noites e `prerelease`
  bloqueando a tag.
- **Mudanças**: uma branch `NNN-slug` por feature, PR obrigatório para a `main` com a CI como
  portão, Conventional Commits e merge sempre humano.

## Governança

- Esta constituição prevalece sobre as demais práticas do repositório. Todo plano MUST passar pelo
  Constitution Check dos princípios I a X, e todo PR MUST verificar a conformidade.
- **Complexidade**: desvio de um princípio só é aceito se registrado na tabela Complexity Tracking
  do plano, com o motivo e a alternativa mais simples rejeitada. Os princípios VII (segredo) e IX
  (evolução só por adição) não admitem desvio.
- **Emendas**: proposta por PR que altera este arquivo, com o Sync Impact Report atualizado e
  os templates dependentes sincronizados. A aprovação é do mantenedor (Guilherme Giusti).
- **Versionamento** (semver): MAJOR para remoção ou redefinição incompatível de princípio ou da
  governança; MINOR para princípio ou seção nova, ou orientação materialmente ampliada; PATCH para
  esclarecimento, redação e correção de typo.
- **Revisão de conformidade**: a cada nova spec e a cada release, confere-se o orçamento (VIII) e o
  escopo da v1 contra esta constituição. Orientação de uso no dia a dia está em `CLAUDE.md`,
  `README.md` e `specs/README.md`.

**Version**: 1.0.0 | **Ratified**: 2026-10-24 | **Last Amended**: 2026-10-24
