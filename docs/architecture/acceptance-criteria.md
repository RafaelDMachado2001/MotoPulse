# Critérios de aceite

F0 é aceito quando os quatro documentos de arquitetura, ADR 0002, `.gitignore` e esqueleto existem; cada requisito tem fase/teste; cada decisão tem dono e reversibilidade; um implementador consegue criar o esquema sem perguntas; nenhum dado real/segredo aparece no histórico. F1 será aceito quando cadastro, manutenção, migrations e reabertura em modo avião preservarem dados.

| Requisitos | Testes |
|---|---|
| RF-01,RF-02,RNF-02 | T0.3,T1.1,T1.6 |
| RF-03,RF-04,RF-10,RNF-06 | T2.1,T2.2,T2.5,T2.6 |
| RF-05,RF-06,RF-07,RNF-03 | T3.1–T3.7,T4.4 |
| RF-08,RF-09 | T4.3,T5.1,T5.2 |
| RNF-04 | T0.6 |

Testes independentes: T0.1 cada requisito tem ID/fase/teste; T0.2 decisões têm dono/reversibilidade; T0.3 esquema sem perguntas; T0.4 cada entregável existe; T0.5 confrontar modelo com CSV real — **pendente até receber CSV real, sem resultado inventado**; T0.6 varrer segredos/dados pessoais no histórico. T1.1–T1.6 cobrem persistência, modo avião, migração, disco cheio, versões e odômetro.
