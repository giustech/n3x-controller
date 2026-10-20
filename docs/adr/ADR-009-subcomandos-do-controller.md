# ADR-009: Subcomandos do binário do controller

Status: Aceito
Data: 2026-10-19

## Contexto
O operador precisa gerar hashes, credenciais de serviço, chaves e valores `ENC[...]` sem depender de ferramentas externas, como o `htpasswd` citado no PRD.

## Decisão
O binário tem os subcomandos `run` (padrão), `validate`, `hash`, `credential new <serviço>`, `encrypt`, `keygen` e `dev-cert`. Segredos são lidos só do stdin. `encrypt` e `keygen` usam o mesmo pacote `core/secret` do `n3x-sidecar`.

## Justificativa
- Fluxo completo de credencial com o próprio N3X
- `validate` permite checar o `n3x.yaml` na CI

## Trade-off
- Mais código para manter no host
- Dependência do pacote de segredos do sidecar (mover para o SDK é pendência)

## Requisitos relacionados
FR-005, FR-009; segurança nos requisitos não funcionais.
