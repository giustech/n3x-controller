# ADR-003: Usuários de leitura e auditoria no plugin core

Status: Substituído parcialmente (2026-10-20)
Data: 2026-10-19

## Contexto
Com o host sem armazenamento (ADR-002), os usuários de leitura (FR-009) e a auditoria (FR-010) precisavam de um dono que persista.

## Decisão
- O plugin core guarda usuários de leitura e auditoria no banco dele e fornece ao host, como capacidades exclusivas:
  - **Diretório de usuários:** confere senha, cria, desativa, remove e redefine; avisa o host quando um usuário é desativado ou removido
  - **Gravador de auditoria:** grava, filtra, exporta (JSON e CSV) e aplica a retenção
- As telas de usuários, auditoria e fluxo por serviço passam para o core
- O host continua dono de: login e logout do painel, sessão, CSRF, bloqueio, root (sempre do arquivo) e montagem dos eventos de auditoria do painel
- A gravação é síncrona: se falhar, a ação do painel é recusada. Exceção de emergência: o login do root é aceito com o evento no log (`audit_fallback=true`)

## Justificativa
- Um único plugin obrigatório concentra os dados persistentes do painel
- Root independente do core garante acesso mesmo com o core falho

## Trade-off
- O core cresce e vira ponto único dos dados do painel
- Exige novas capacidades privilegiadas no contrato do SDK

## Requisitos relacionados
FR-008, FR-009, FR-010, FR-013, FR-016; compliance nos requisitos não funcionais.

## Atualizado em 2026-10-20
Decisões do HLD do `n3x-plugin-core` mudam parte deste ADR na v1:
- Auditoria (FR-010) adiada para a v2 (ADR-002 do `n3x-plugin-core`): o core não fornece o gravador de auditoria, saem a tela de auditoria, a gravação síncrona, a recusa da ação quando a gravação falha e o `audit_fallback=true`. Login, logout e bloqueio do painel ficam só no log JSON operacional do host
- Usuários de leitura vêm só da seção do core no `n3x.yaml` do controller (ADR-003 do `n3x-plugin-core`): o diretório só confere senha; criar, desativar, remover, redefinir e trocar a própria senha pelo painel saem da v1, e o aviso de usuário desativado deixa de ser usado
- O core guarda tudo em memória na v1, sem banco (ADR-001 do `n3x-plugin-core`)
- Segue valendo: usuários de leitura no core, root sempre do arquivo e host dono de login, sessão, CSRF e bloqueio
