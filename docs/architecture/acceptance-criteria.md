# Critérios de aceite

## F0

F0 exige existência dos quatro documentos, ADR 0002, `.gitignore` e esqueleto; coerência ADR 0001; requisitos/testes completos; decisões com dono e reversibilidade; esquema implementável sem perguntas; nenhum dado real/segredo.

## F1

F1 exige criação/edição de moto, manutenção, itens/planos/odômetro, migration transacional, fixture demo, modo avião e reabertura mantendo contagens e hashes.

## Matriz requisito × teste

| Requisito | Testes |
|---|---|
| RF-01 | T1.1 |
| RF-02 | T1.2 |
| RF-03 | T1.3 |
| RF-04 | T1.4 |
| RF-05 | T1.5 |
| RF-06 | T1.6 |
| RF-07 | T2.1 |
| RF-08 | T2.2 |
| RF-09 | T2.3 |
| RF-10 | T3.1 |
| RF-11 | T3.2 |
| RF-12 | T3.3 |
| RF-13 | T4.1 |
| RF-14 | T5.1 |
| RF-15 | T7.1 |
| RNF-01 | T1.7 |
| RNF-02 | T0.3 |
| RNF-03 | T3.4 |
| RNF-04 | T0.6 |
| RNF-05 | T6.1 |
| RNF-06 | T1.8 |
| RNF-07 | T1.9 |
| RF-16 | T3.5 |
| RF-17 | T1.10 |
| RF-18 | T2.4 |
| RF-19 | T3.6 |
| RF-20 | T5.2 |

## Testes definidos

Cada teste abaixo tem pré-condição, passos e resultado esperado.

- **T0.1** — Pré: docs presentes. Liste IDs e fase/teste; esperado: todos têm campos.
- **T0.2** — Pré: ADRs. Liste decisões; esperado: cada uma tem dono e reversibilidade.
- **T0.3** — Pré: data-model. Criar DDL em fixture; esperado: tipos, NULL/default, CHECK/FK/ON DELETE sem pergunta.
- **T0.4** — Pré: branch. Verificar cada entregável; esperado: todos existem.
- **T0.5** — Pré: CSV real disponível fora do Git. Confrontar colunas; esperado: pendente enquanto ausente, sem resultado inventado.
- **T0.6** — Pré: histórico Git. Buscar segredos, CSV/GPX/coordenadas reais; esperado: nenhum; fixture sintética não falha.
- **T1.1** — Pré: DB vazio. Criar/editar moto, fechar/abrir; esperado: valores persistem.
- **T1.2** — Pré: moto. Criar manutenção com e sem custo; esperado: centavos/moeda coerentes e NULL motivado.
- **T1.3** — Pré: registro. Adicionar itens; esperado: vínculo e cascade corretos.
- **T1.4** — Pré: moto. Criar plano e vincular registro; esperado: plan_item/record_plan preservam histórico.
- **T1.5** — Pré: leituras. Inserir regressão; esperado: só correção explícita passa.
- **T1.6** — Pré: fixture demo. Abrir offline; esperado: dados aparecem sem rede.
- **T1.7** — Pré: migration. Interromper transação; esperado: sem estado parcial e integrity_check passa.
- **T1.8** — Pré: registros persistidos. Matar/reabrir; esperado: contagens, hashes e registros iguais.
- **T1.9** — Pré: modo avião. Usar capacidades listadas; esperado: nenhuma depende de rede.
- **T2.1** — Pré: permissão. Exercitar start/pause/resume/finish; esperado: estados e segmentos corretos.
- **T2.2** — Pré: recording. Matar processo/reabrir; esperado: interrupted recuperável sem apagar pontos.
- **T2.3** — Pré: pontos válidos/inválidos. Calcular distância; esperado: só válidos, sem interpolar.
- **T3.1** — Pré: fixture CSV sintética. Importar; esperado: bytes/hash/origem/issues persistidos.
- **T3.2** — Pré: viagem e sessão. Associar em ambos sentidos e omitir; esperado: todas combinações válidas.
- **T3.3** — Pré: amostras sintéticas. Calcular métricas; esperado: versão, chave, distância GPS/roda/Rideology separadas.
- **T3.4** — Pré: import. Recalcular; esperado: bruto/hash iguais e derivado novo versionado.
- **T4.1** — Pré: findings. Gerar quatro classes; esperado: cada uma com evidência e classificação.
- **T5.1** — Pré: métricas. Gerar relatório; esperado: versão, input_hash, evidência e incerteza.
- **T6.1** — Pré: adaptador BLE futuro. Inspecionar interface; esperado: nenhuma operação de escrita em ECU/ABS/KTRC/painel.
- **T7.1** — Pré: produto com dados. Exportar/importar manualmente; esperado: conteúdo e hashes verificáveis (fase ainda aberta).

Testes fortes adicionais: **T1.11** matar processo imediatamente após salvar; **T1.12** instalação limpa, modo avião e reboot; **T1.13** migrar de cada versão anterior; **T1.14** disco cheio; **T1.15** demo não contamina dados reais; **T2.4** revogar permissão e verificar interrupted; **T2.5** injetar salto de 10 km e rejeitar; **T2.6** pausar/retomar 20 vezes sem duplicar; **T2.7** alterar relógio +/-1 h; **T3.5** remover cada coluna e marcar ausente; **T3.6** arquivo grande não trava UI; **T3.7** comparar métrica com cálculo independente; **T4.2** histórico 0/1/N viagens; **T4.3** anomalia sintética e falsos positivos; **T4.4** algoritmo versionado preserva resultado; **T4.5** sensor constante; **T5.2** citação inexistente/número fora da entrada rejeitados; **T5.3** injeção em nota; **T5.4** offline degrada; **T5.5** payload sem coordenadas sem consentimento; **T7.2** exportação reimportável; **T7.3** exclusão remove rastros; **T7.4** logs sem coordenadas. Cada teste tem pré-condição, passos e resultado esperado análogos aos testes definidos acima.

### T0.5 — confronto executado

Confrontado com `csv-structure.md`, fatos de um único arquivo real, sem abrir/copiar o CSV:

| Coluna/fato | Modelo | Resultado |
|---|---|---|
| elapsed_msec | telemetry_sample.elapsed_ms | representado |
| gps_latitude/longitude | telemetry_sample.latitude/longitude | representado, decimal por coluna |
| instant_fuel_consumption(km/L) | fuel_km_per_l | preservado; 0.0/99.9 [A VERIFICAR] |
| water_temperature | values_json/derivado | preservado como lido |
| boost_temperature, acceleration, throttle_position, boost_pressure, lean_angle | values_json | constantes preservadas |
| engine_RPM, wheel_speed, gear_position, rideology_score | rpm, speed_mps, gear, values_json | N representável |
| preâmbulo Title/Comment/Distance/Time/Vehicle Nickname | telemetry_import.metadata_json | representado; UserAccount Name só bruto local |
| tempo sem fuso e lacuna | telemetry_session unknown/precision; elapsed_ms | representado |

Resultado: T0.5 passa estruturalmente contra este único arquivo; validação de valores/colunas adicionais continua limitada ao relatório fornecido, sem inventar dados.

Coerência automatizável: `python3 - <<'PY'` (ou script equivalente) deve extrair RF/RNF de requirements, extrair a primeira coluna da matriz e comparar conjuntos; deve também extrair tokens `T[0-9]+\.[0-9]+` de requirements e confirmar que cada um aparece como cabeçalho/definição nesta seção. Falha se houver diferença.
