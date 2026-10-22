# Specs do n3x-controller

Índice das specs (Spec Kit) e das FDDs que as originam. Cada spec nasce de uma FDD: a FDD fixa o comportamento verificável e a spec organiza a implementação (`spec.md`, `plan.md`, `tasks.md`).

Documentos de base: [PRD](../docs/prd/PRD-N3X.md) · [HLD](../docs/HLD.md) · [ADRs](../docs/adr/)

## Índice

| # | Spec | FDD | Resumo |
|---|------|-----|--------|
| 01 | [01_fundacao_do_host](./01_fundacao_do_host/) | [FDD-CTL-01](../docs/fdd/FDD-CTL-01-fundacao-do-host.md) ([json](../docs/fdd/FDD-CTL-01-fundacao-do-host.json)) | `n3x.yaml`, `host.Run`, CLI e operação |
| 02 | [02_verificador_de_credencial](./02_verificador_de_credencial/) | [FDD-CTL-02](../docs/fdd/FDD-CTL-02-verificador-de-credencial.md) ([json](../docs/fdd/FDD-CTL-02-verificador-de-credencial.json)) | Verificação de credencial de serviço (bcrypt, contenção, bloqueio) |
| 03 | [03_host_de_plugins](./03_host_de_plugins/) | [FDD-CTL-03](../docs/fdd/FDD-CTL-03-host-de-plugins.md) ([json](../docs/fdd/FDD-CTL-03-host-de-plugins.json)) | Transforma o `Bundle` do pré-voo em plugins vivos e protegidos |
| 04 | [04_servidor_grpc](./04_servidor_grpc/) | [FDD-CTL-04](../docs/fdd/FDD-CTL-04-servidor-grpc.md) ([json](../docs/fdd/FDD-CTL-04-servidor-grpc.json)) | Servidor gRPC único: porta, TLS e autenticação |
| 05 | [05_sessao_e_login_do_painel](./05_sessao_e_login_do_painel/) | [FDD-CTL-05](../docs/fdd/FDD-CTL-05-sessao-do-painel-e-login.md) | Sessão do painel, root e login |
| 06 | [06_painel_layout_e_slots](./06_painel_layout_e_slots/) | [FDD-CTL-06](../docs/fdd/FDD-CTL-06-painel-layout-e-slots.md) | Layout do painel, visão geral, slots de plugin e acessibilidade |
| 07 | [07_simulador_de_carga](./07_simulador_de_carga/) | [FDD-CTL-07](../docs/fdd/FDD-CTL-07-simulador-de-500-sidecars.md) | Simulador de 500 sidecars e orçamento do controller |

## Ordem e dependências

As FDDs se apoiam nas anteriores; implemente na ordem do índice.

```
01 fundação ──> 02 credencial ──> 03 host de plugins ──> 04 servidor gRPC
                                         │                      │
                                         └──> 05 sessão/login ──> 06 painel
                                                                  │
                         07 simulador (valida 01–06 contra as metas do PRD)
```

## Convenções

- Cada spec vive em `specs/NN_nome/`, e o `NN` e o código `CTL-NN` da FDD coincidem.
- A FDD é a fonte de verdade do comportamento. Se a spec divergir, corrija a spec ou abra revisão da FDD.
- Ao criar ou concluir uma spec, atualize a tabela acima.
