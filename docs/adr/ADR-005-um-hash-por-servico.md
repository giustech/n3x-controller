# ADR-005: Um único hash por serviço

Status: Aceito
Data: 2026-10-19

## Contexto
O HLD de plataforma deixou pendente aceitar vários hashes por serviço durante a rotação da credencial.

## Decisão
- Cada serviço tem um único `secret_hash` no `n3x.yaml` do controller
- Rotação: gerar a credencial nova (`credential new`), trocar o hash, reiniciar o controller e redeployar o serviço

## Justificativa
- Schema e verificação mais simples
- Uma falha de login custa só um bcrypt

## Trade-off
- Durante a troca, os sidecars do serviço falham no login e somem do painel até o redeploy; o tráfego não é afetado

## Requisitos relacionados
FR-005; segurança nos requisitos não funcionais.
