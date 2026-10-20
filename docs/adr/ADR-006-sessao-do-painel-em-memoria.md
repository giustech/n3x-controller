# ADR-006: Sessão do painel só em memória

Status: Aceito
Data: 2026-10-19

## Contexto
O host não tem armazenamento (ADR-002). A sessão do painel poderia ficar em memória ou num cookie assinado.

## Decisão
- Sessão em memória, com ID aleatório de 32 bytes no cookie `n3x_session` (`HttpOnly`, `Secure`, `SameSite=Strict`)
- Expira após 30 min sem uso ou 12 h no total
- Desativar ou remover um usuário corta as sessões dele na hora

## Justificativa
- Revogação imediata, sem consulta ao core a cada requisição
- Nada para persistir

## Trade-off
- Restart do controller obriga todos a fazer login de novo
- Impede mais de uma réplica no futuro sem rever a decisão

## Requisitos relacionados
FR-008, FR-009; segurança nos requisitos não funcionais.

## Atualizado em 2026-10-20
Na v1 não há gestão de usuários pelo painel (ADR-003 do `n3x-plugin-core`): desativar ou remover um usuário é editar a seção do core no `n3x.yaml` e reiniciar o controller, o que já descarta todas as sessões. O corte de sessões por aviso do core volta a valer quando a gestão em runtime voltar.
