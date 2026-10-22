# ADRs do n3x-controller

Decisões de arquitetura registradas no HLD do controller (`../HLD.md`). A numeração é própria deste repositório; ADRs de outros repositórios são citados com o nome dele (por exemplo, "ADR-003 do `n3x-sdk`"). Cada ADR traz contexto, decisão, justificativa, trade-off e requisitos relacionados do PRD (`../prd/PRD-N3X.md`).

| ADR | Título | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-host-como-biblioteca.md) | Host como biblioteca, com o binário sempre gerado pelo `n3x-plugin build` | Aceito |
| [ADR-002](ADR-002-controller-sem-armazenamento.md) | Controller sem armazenamento, cada plugin traz o próprio banco | Aceito (substitui o ADR-002 do `n3x-sdk`) |
| [ADR-003](ADR-003-usuarios-e-auditoria-no-core.md) | Usuários de leitura e auditoria no plugin core | Substituído parcialmente em 2026-10-20 (ADR-001, ADR-002 e ADR-003 do `n3x-plugin-core`) |
| [ADR-004](ADR-004-verificador-de-credencial-no-host.md) | Verificador de credencial no host, com cache de 1 h | Aceito (atualizado em 2026-10-20) |
| [ADR-005](ADR-005-um-hash-por-servico.md) | Um único hash por serviço | Aceito |
| [ADR-006](ADR-006-sessao-do-painel-em-memoria.md) | Sessão do painel só em memória | Aceito (atualizado em 2026-10-20) |
| [ADR-007](ADR-007-grpc-so-para-sidecars.md) | Servidor gRPC só para sidecars na v1 | Aceito |
| [ADR-008](ADR-008-polling-e-cartoes.md) | Painel com polling de 5 s e visão geral por cartões | Aceito |
| [ADR-009](ADR-009-subcomandos-do-controller.md) | Subcomandos do binário do controller | Aceito (atualizado em 2026-10-22) |
