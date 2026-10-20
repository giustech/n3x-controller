# ADR-008: Painel com polling de 5 s e visão geral por cartões

Status: Aceito
Data: 2026-10-19

## Contexto
O PRD deixou em aberto a atualização ao vivo (polling ou SSE). Com os dados nos plugins (ADR-002 e ADR-003), a visão geral também precisava de uma forma de reunir informação de vários plugins.

## Decisão
- Blocos de status atualizados por polling htmx (`hx-trigger="every 5s"`), com o intervalo em `panel.refresh_interval`; respostas parciais em HTML
- A visão geral do host é montada com cartões que os plugins registram; o plugin core fornece os cartões de sidecars por estado, divergências e alertas
- SSE fica como evolução futura

## Justificativa
- Sem conexões abertas nem barramento de eventos no host
- Funciona atrás de qualquer proxy e cabe na meta de p95 < 300 ms

## Trade-off
- Atraso de até 5 s e requisições mesmo sem mudança

## Requisitos relacionados
FR-008, FR-011; desempenho nos requisitos não funcionais.
