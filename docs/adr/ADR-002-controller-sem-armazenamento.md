# ADR-002: Controller sem armazenamento, cada plugin traz o próprio banco

Status: Aceito (substitui o ADR-002 do `n3x-sdk`)
Data: 2026-10-19

## Contexto
O PRD deixou o armazenamento do controller para o HLD, e o ADR-002 do `n3x-sdk` previa dois modos no controller (efêmero ou Postgres), com migrations isoladas por plugin. Isso obrigaria o host a oferecer uma abstração de dados que funcionasse nos dois modos e a gerenciar migrations de terceiros.

## Decisão
- O host do controller **não tem armazenamento**: nem banco, nem migrations, nem modo degradado próprio
- Cada plugin que precisa guardar dados traz a própria implementação e a própria conexão, configuradas na seção dele em `plugins` do `n3x.yaml` do controller, com segredos em `ENC[...]`
- Um plugin acessa dados de outro só pela API pública dele
- O host oferece um canal de estado (`ok`, `degradado`, `falho`) para cada plugin, mostrado como banner no painel

## Justificativa
- Host menor e sem dependência de banco
- Isolamento natural: cada plugin tem conexão e credenciais próprias
- Cada plugin escolhe o banco adequado ao que guarda

## Trade-off
- Cada plugin reimplementa conexão, migrations e retenção
- "Controller degradado" (FR-006, FR-008) vira estado por plugin
- Exige atualizar o PRD (FR-006, FR-008, FR-011 e escopo) e o HLD de plataforma

## Requisitos relacionados
FR-006, FR-007, FR-008, FR-011; confiabilidade nos requisitos não funcionais.
