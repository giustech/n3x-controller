# ADR-007: Servidor gRPC só para sidecars na v1

Status: Aceito
Data: 2026-10-19

## Contexto
O servidor gRPC poderia aceitar também os papéis do painel (root e leitura), por exemplo para uma CLI de administração.

## Decisão
- Na v1, todo método gRPC exige o papel `sidecar`; root e leitura usam só o painel HTTPS
- Plugins não declaram papel por método na v1

## Justificativa
- Contrato de plugin mais simples
- Nenhum login de usuário no gRPC

## Trade-off
- Uma CLI de administração futura exigirá papéis por método no contrato do SDK

## Requisitos relacionados
FR-005, FR-011, FR-013.
