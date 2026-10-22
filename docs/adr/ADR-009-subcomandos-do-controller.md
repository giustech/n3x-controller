# ADR-009: Subcomandos do binário do controller

Status: Aceito
Data: 2026-10-19

## Contexto
O operador precisa gerar hashes, credenciais de serviço, chaves e valores `ENC[...]` sem depender de ferramentas externas, como o `htpasswd` citado no PRD.

## Decisão
O binário tem os subcomandos `run` (padrão), `validate`, `hash`, `credential new <serviço>`, `encrypt`, `keygen`, `dev-cert`, `probe`, `version` e `help`. Segredos são lidos só do stdin. `encrypt` e `keygen` usam o pacote `secret` do `n3x-sdk` (ADR-014 do `n3x-sdk`).

## Justificativa
- Fluxo completo de credencial com o próprio N3X
- `validate` permite checar o `n3x.yaml` na CI

## Trade-off
- Mais código para manter no host
- Dependência do pacote `secret` do SDK (a pendência de mover o pacote foi resolvida pelo ADR-014 do `n3x-sdk`)

## Requisitos relacionados
FR-005, FR-009; segurança nos requisitos não funcionais.

## Atualizado em 2026-10-22
- Acrescentados `probe` (healthcheck de `/live` ou `/ready` em `127.0.0.1:17091`, necessário porque a imagem distroless não tem shell), `version` e `help`, definidos na FDD-CTL-01
- O pacote de segredos é o `secret` do `n3x-sdk`, e não mais o `core/secret` do `n3x-sidecar`
