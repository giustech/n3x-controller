# ADR-004: Verificador de credencial no host, com cache de 1 h

Status: Aceito
Data: 2026-10-19

## Contexto
O login dos sidecars confere o segredo com bcrypt (custo ≥ 12, cerca de 250 ms de CPU). Após um restart sem chave HMAC em arquivo, os 500 sidecars fazem login juntos: cerca de 2 minutos de CPU cheia em 1 vCPU, contra a meta de voltar ao painel em 30 s. O login também é a única rota pública.

## Decisão
- A rota `Login` é do core, mas a conferência do segredo é do host, por um verificador oferecido só ao core
- Os hashes ficam em `services.<nome>.secret_hash` no `n3x.yaml` do controller
- No máximo 2 bcrypt simultâneos, com fila curta e prazo; sem vaga, `RESOURCE_EXHAUSTED`
- Bloqueio após 5 falhas por IP e por serviço, por 15 min
- Cache de 1 h de credenciais verificadas: guarda só um HMAC-SHA256 do segredo com chave aleatória do processo; falhas não entram

## Justificativa
- Os pods de um serviço compartilham o segredo: com 50 serviços, são 50 bcrypt em vez de 500
- Proteção contra força bruta e DoS num só lugar

## Trade-off
- Uma impressão do segredo fica em memória por até 1 h
- Trocar o hash exige restart, o que também limpa o cache

## Requisitos relacionados
FR-002, FR-005, FR-016; desempenho e segurança nos requisitos não funcionais.

## Atualizado em 2026-10-20
A chave HMAC dos tokens passou a ser do plugin core (ADR-004 do `n3x-plugin-core`): onde se lê "chave HMAC em arquivo", leia `session.hmac_key` na seção do core. Sem essa chave, o restart invalida todos os tokens e todos refazem o login; com o cache por serviço, a meta de 30 s vale até cerca de 200 serviços.
