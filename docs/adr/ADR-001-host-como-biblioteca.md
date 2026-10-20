# ADR-001: Host como biblioteca, com o binário sempre gerado pelo `n3x-plugin build`

Status: Aceito
Data: 2026-10-19

## Contexto
As metades de controller dos plugins são código Go, e Go não carrega plugins em runtime. Para um plugin aparecer no controller, ele precisa ser compilado no mesmo binário. O FR-015 já prevê que o binário saia da imagem `controller-builder`. Era preciso decidir se o repositório também publica um binário pronto.

## Decisão
- O repositório publica o módulo Go `host/` e a imagem base `n3x/controller:<versão>`, sem binário
- O binário sai sempre do `n3x-plugin build`, que gera um `main` chamando `host.Run(core.Plugin(), <plugins>...)`
- Para desenvolvimento e CI, o repositório mantém `cmd/n3x-controller-dev` (host + plugin core) e um `go.work` com `n3x-controller`, `n3x-plugin-core` e `n3x-sdk`
- Testes em três níveis: unitários sem Docker (plugins falsos, `bufconn`, `httptest`), integração ponta a ponta com docker compose em `n3x-sdk/deploy/` e kit de testes do SDK para autores de plugin

## Justificativa
- Um único caminho para obter o controller, com a mesma lista de plugins do sidecar
- Mesmo modelo do sidecar (ADR-001 do `n3x-sidecar`)

## Trade-off
- Mesmo quem usa só o plugin core precisa de um Dockerfile com o builder
- O `main` de desenvolvimento precisa acompanhar o `main` gerado pelo builder

## Requisitos relacionados
FR-011, FR-015.
