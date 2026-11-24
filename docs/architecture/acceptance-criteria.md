# Critérios de aceite

## F0

| Critério | Verificação |
|---|---|
| Todos os entregáveis do plano mestre existem | T0.4 |
| Todo requisito tem fase e teste, e a matriz coincide nos dois sentidos | T0.1 |
| Toda decisão tem dono e reversibilidade; irreversíveis estão marcadas | T0.2 |
| O esquema é criado do documento, sem adaptação, e resiste à bateria de casos | T0.3 |
| O modelo representa a estrutura conhecida do CSV | T0.5 |
| Nenhum dado real ou segredo no histórico | T0.6 |
| Divergências entre as duas análises independentes estão consolidadas | tabela do ADR 0002 |
| Documentos coerentes com o ADR 0001 | revisão por pessoa diferente do autor |

O critério do ADR 0001 "existe uma fixture sintética com a mesma estrutura de colunas" está **atribuído à F1 (T1.18) e ainda não foi cumprido**. A F0 não o declara atendido.

## F1

| Critério | Verificação |
|---|---|
| Criar e editar a moto e reabrir em modo avião | T1.2, T1.8, T1.15 |
| Registrar manutenção, itens, planos e odômetro com as regras do modelo | T1.6, T1.9, T1.10, T1.11, T1.12 |
| Nenhum registro confirmado se perde | T1.1, T1.4, T1.17 |
| Migrações transacionais | T1.3, T1.5 |
| Demonstração sintética e isolada | T1.7, T1.13, T1.18 |
| Navegação entre as áreas | T1.14 |
| Formatos e ausência garantidos | T1.16 |

A F1 só começa depois dos spikes S1 a S3 e das decisões abertas com prazo na F1.

## Matriz requisito x teste

| Requisito | Testes |
|---|---|
| RF-01 | T1.8 |
| RF-02 | T1.9 |
| RF-03 | T1.10 |
| RF-04 | T1.11 |
| RF-05 | T1.6, T1.12 |
| RF-06 | T1.7, T1.13 |
| RF-07 | T2.6, T2.8 |
| RF-08 | T2.1, T2.2, T2.3 |
| RF-09 | T2.5, T2.7, T2.10 |
| RF-10 | T3.1, T3.2, T3.3, T3.8 |
| RF-11 | T3.9 |
| RF-12 | T3.4, T3.7, T3.10 |
| RF-13 | T4.1, T4.2, T4.3, T4.4, T4.5, T4.6 |
| RF-14 | T5.1, T5.2, T5.6 |
| RF-15 | T7.2, T7.5 |
| RF-16 | T3.14 |
| RF-17 | T1.14 |
| RF-18 | T2.11 |
| RF-19 | T2.12 |
| RF-20 | T3.13 |
| RF-21 | T5.7 |
| RF-22 | T5.8 |
| RF-23 | T3.11 |
| RF-24 | T3.12 |
| RF-25 | T2.14, T3.17 |
| RF-26 | T6.3, T6.4 |
| RF-27 | T3.18 |
| RNF-01 | T1.4, T1.17, T3.5 |
| RNF-02 | T0.3, T1.16 |
| RNF-03 | T3.1, T3.15 |
| RNF-04 | T0.6, T1.18 |
| RNF-05 | T6.1, T6.2 |
| RNF-06 | T1.1 |
| RNF-07 | T1.2, T1.15 |
| RNF-08 | T2.14, T3.17 |
| RNF-09 | T3.16, T7.4 |
| RNF-10 | T1.3, T1.5 |
| RNF-11 | T3.6 |
| RNF-12 | T2.4 |
| RNF-13 | T5.3, T5.4, T5.5 |
| RNF-14 | T7.1, T7.3 |
| RNF-15 | T0.1, T0.2, T0.4, T0.5 |

## Definição dos testes

Os IDs da análise independente mantêm o significado original. Testes próprios deste contrato continuam a numeração da fase; T2.9 e T2.13 não são usados. O teste pertence à fase do primeiro número, salvo indicação no resultado.

### F0

**T0.1** — Rastreabilidade

- Pré-condição: documentos da F0 presentes.
- Passos: extrair todo RF e RNF e conferir fase e teste.
- Resultado esperado: todos têm fase e ao menos um teste definido, e a matriz coincide com os requisitos.

**T0.2** — Decisões

- Pré-condição: ADR 0002 presente.
- Passos: listar as decisões aceitas, propostas e abertas.
- Resultado esperado: cada uma tem dono e reversibilidade da escala definida, e as irreversíveis estão marcadas.

**T0.3** — Esquema sem perguntas

- Pré-condição: SQLite 3.37 ou superior e diretório temporário.
- Passos: extrair os blocos `sql` de `data-model.md`, executar sem adaptação e rodar a bateria de inserções válidas e inválidas.
- Resultado esperado: o esquema é criado e cada caso tem o resultado esperado.

**T0.4** — Entregáveis

- Pré-condição: branch da F0.
- Passos: conferir a existência de cada entregável do plano mestre para a F0.
- Resultado esperado: todos existem.

**T0.5** — Confronto com a estrutura do CSV

- Pré-condição: relatório de estrutura do CSV, sem abrir o arquivo real.
- Passos: comparar cada coluna e metadado com o modelo.
- Resultado esperado: todo campo é representável, ou fica fora do banco por decisão registrada; a fixture sintética do ADR 0001 não é avaliada aqui, porque pertence à F1 (T1.18).

**T0.6** — Dados reais e segredos

- Pré-condição: histórico da branch.
- Passos: buscar CSV, GPX, chaves, coordenadas e nomes reais em todos os commits.
- Resultado esperado: nada encontrado, e fixtures sintéticas não são acusadas.

### F1

**T1.1** — Morte após salvar

- Pré-condição: banco vazio.
- Passos: salvar moto e manutenção, matar o processo logo após a confirmação e reabrir.
- Resultado esperado: os dois registros estão íntegros.

**T1.2** — Instalação limpa sem rede

- Pré-condição: aparelho em modo avião, sem o aplicativo.
- Passos: instalar, abrir, cadastrar, reiniciar o aparelho e reabrir.
- Resultado esperado: tudo funciona e os dados permanecem.

**T1.3** — Morte durante migração

- Pré-condição: banco da versão anterior com dados.
- Passos: iniciar a migração e matar o processo em cada passo.
- Resultado esperado: ao reabrir, o banco está inteiro na versão antiga ou na nova e `integrity_check` é `ok`.

**T1.4** — Disco cheio

- Pré-condição: aparelho com espaço esgotado.
- Passos: salvar um registro.
- Resultado esperado: erro `storage_full` visível, nada parcial e dados antigos intactos.

**T1.5** — Migração de cada versão

- Pré-condição: um banco de exemplo por versão de esquema.
- Passos: migrar cada um para a atual.
- Resultado esperado: contagens e valores preservados e `integrity_check` é `ok`.

**T1.6** — Odômetro regressivo

- Pré-condição: leitura de 1.000.000 m.
- Passos: inserir leitura posterior menor, sem correção.
- Resultado esperado: recusada com `odometer_regression`.

**T1.7** — Demonstração não contamina

- Pré-condição: moto real e moto de demonstração com dados.
- Passos: consultar totais e listas reais; tentar gravar viagem de demonstração na moto real.
- Resultado esperado: totais reais não mudam e a gravação é recusada.

**T1.8** — Moto persiste

- Pré-condição: banco vazio.
- Passos: criar com e sem apelido digitado, editar, fechar e reabrir.
- Resultado esperado: os valores editados permanecem e o apelido é opcional.

**T1.9** — Custo e moeda

- Pré-condição: moto cadastrada.
- Passos: gravar com custo e moeda, sem custo com motivo, custo sem moeda, moeda sem custo, custo negativo e moeda `reais`.
- Resultado esperado: os dois primeiros são aceitos e os demais recusados.

**T1.10** — Itens de manutenção

- Pré-condição: registro de manutenção existente.
- Passos: adicionar dois itens, um ligado a item de plano, e tentar excluir o registro.
- Resultado esperado: itens vinculados corretamente e exclusão recusada por RESTRICT.

**T1.11** — Planos

- Pré-condição: duas motos, um plano na primeira.
- Passos: vincular o plano a um registro da primeira e a um registro da segunda; criar plano sem intervalo.
- Resultado esperado: só o primeiro vínculo é aceito.

**T1.12** — Correção de odômetro

- Pré-condição: leitura errada de 1.000.000 m.
- Passos: corrigir para 500.000 m com nota, corrigir sem nota, corrigir leitura de outra moto e editar leitura.
- Resultado esperado: só a correção com nota é aceita e a original permanece consultável.

**T1.13** — Demonstração sem rede

- Pré-condição: modo avião.
- Passos: abrir os dados de demonstração.
- Resultado esperado: dados sintéticos aparecem, identificados como demonstração.

**T1.14** — Navegação

- Pré-condição: aplicativo aberto.
- Passos: ir e voltar entre garagem, manutenção, viagens e relatórios.
- Resultado esperado: as quatro áreas abrem e o retorno preserva o contexto.

**T1.15** — Capacidades sem rede

- Pré-condição: modo avião e fixtures.
- Passos: exercitar cada capacidade local disponível na fase.
- Resultado esperado: nenhuma pede rede; repetir o teste em F2, F3 e F4 com o que cada uma acrescenta.

**T1.16** — Formatos

- Pré-condição: repositórios ligados a banco temporário.
- Passos: gravar ID que não é UUID, instante inválido, offset inválido, JSON inválido e medição ausente sem motivo.
- Resultado esperado: todos são recusados.

**T1.17** — Conexão e atomicidade

- Pré-condição: banco temporário.
- Passos: abrir conexões novas e ler `foreign_keys`, `recursive_triggers` e `journal_mode`; falhar no meio de uma transação.
- Resultado esperado: valores ligados, WAL ativo e nada parcial visível.

**T1.18** — Fixture sintética

- Pré-condição: repositório da F1.
- Passos: conferir as 14 colunas, a procedência gerada por script e a ausência de dados reais.
- Resultado esperado: a fixture existe e tem a estrutura documentada, sem valor copiado de arquivo real.

### F2

**T2.1** — Morte em cada estado

- Pré-condição: viagem em cada estado não final.
- Passos: matar o processo à força e reabrir.
- Resultado esperado: perda de até 5 s [PROPOSTA] e estado conforme a tabela.

**T2.2** — Reinício na viagem

- Pré-condição: viagem em gravação.
- Passos: reiniciar o aparelho e reabrir o aplicativo.
- Resultado esperado: viagem `interrupted`, pontos gravados intactos e recuperação oferecida.

**T2.3** — Permissão revogada

- Pré-condição: viagem em gravação e outra pausada.
- Passos: revogar a permissão em cada caso.
- Resultado esperado: gravando vira `interrupted`; pausada permanece `paused` e `resume` é recusado; nada é apagado.

**T2.4** — Tela apagada em economia

- Pré-condição: aparelho de referência em modo economia.
- Passos: gravar 2 h com tela apagada.
- Resultado esperado: metas do spike S1 para lacunas e bateria atendidas.

**T2.5** — Salto de 10 km

- Pré-condição: trajeto sintético.
- Passos: injetar um ponto a 10 km.
- Resultado esperado: ponto `suspect` por `jump`, guardado e fora da distância.

**T2.6** — Pausar e retomar 20 vezes

- Pré-condição: viagem em gravação.
- Passos: alternar 20 vezes.
- Resultado esperado: um segmento por retomada, sem ponto duplicado e sem distância entre segmentos.

**T2.7** — Relógio alterado

- Pré-condição: viagem em gravação.
- Passos: adiantar e atrasar o relógio em 1 h.
- Resultado esperado: duração e ordem seguem o relógio monotônico e a distância não muda.

**T2.8** — Estados e transições

- Pré-condição: uma viagem por estado.
- Passos: aplicar todos os eventos da tabela em todos os estados.
- Resultado esperado: só as transições listadas mudam o estado; as demais devolvem `invalid_transition`, e o banco as recusa mesmo em acesso direto; `finished` e `cancelled` são finais.

**T2.10** — Distância e cobertura

- Pré-condição: trajeto sintético com trechos válido, sem precisão, congelado e sem sinal.
- Passos: avaliar e calcular.
- Resultado esperado: soma só os pares válidos, igual ao cálculo manual, com `covered_ms` e `excluded_ms` exibidos.

**T2.11** — Amostragem adaptativa

- Pré-condição: trajeto sintético com movimento e parada.
- Passos: gravar.
- Resultado esperado: intervalos seguem a política configurada e nenhum início de movimento é perdido além de 5 s [PROPOSTA].

**T2.12** — Paradas

- Pré-condição: trajeto com paradas de 30, 60 e 120 s.
- Passos: classificar.
- Resultado esperado: só as de 60 e 120 s são paradas [PROPOSTA], com início e fim em `evidence_json`.

**T2.14** — Exclusão de pontos

- Pré-condição: viagem encerrada com pontos e outra viagem.
- Passos: tentar alterar, apagar e substituir um ponto; repetir com autorização da primeira.
- Resultado esperado: sem autorização tudo é recusado; com ela só os pontos da primeira e seus derivados são apagados, ponto novo e reabertura são recusados e o registro fica permanente.

### F3

**T3.1** — Hash do original

- Pré-condição: fixture sintética.
- Passos: importar, calcular métricas, reabrir e recalcular o SHA-256 do arquivo bruto.
- Resultado esperado: o hash é igual ao do original e ao registrado.

**T3.2** — Reimportar e renomear

- Pré-condição: fixture já importada.
- Passos: importar de novo, com o mesmo nome e com outro.
- Resultado esperado: zero importações, sessões e amostras novas.

**T3.3** — Matriz de formatos

- Pré-condição: fixtures de encoding, delimitador e decimal do spike S2.
- Passos: importar cada combinação.
- Resultado esperado: valores idênticos, ou recusa com motivo.

**T3.4** — Coluna removida

- Pré-condição: uma fixture por coluna ausente.
- Passos: importar e calcular.
- Resultado esperado: campo ausente com motivo, métrica dependente ausente e nunca 0; sem `elapsed`, o arquivo é recusado.

**T3.5** — Morte durante importação

- Pré-condição: fixture grande.
- Passos: matar o processo em cada etapa e reabrir.
- Resultado esperado: nada parcial visível e nova tentativa funciona.

**T3.6** — Arquivo grande

- Pré-condição: fixture sintética de 60.000 linhas.
- Passos: importar interagindo com a tela.
- Resultado esperado: nenhum bloqueio da interface acima de 500 ms [PROPOSTA] e cancelamento funciona.

**T3.7** — Cálculo independente

- Pré-condição: fixture com resultado conhecido.
- Passos: comparar cada métrica com um cálculo feito fora do aplicativo.
- Resultado esperado: diferença relativa de até 0,1% [PROPOSTA].

**T3.8** — Fidelidade e problemas

- Pré-condição: fixture CP932 com preâmbulo, N, km/L 0.0 e 99.9, linha truncada e cabeçalho estranho.
- Passos: importar e comparar `raw_values_json` com o arquivo.
- Resultado esperado: texto de cada célula idêntico; problemas com linha ou escopo de arquivo; 0.0 e 99.9 preservados.

**T3.9** — Vínculo opcional

- Pré-condição: viagem e sessão da mesma moto, e viagem de outra.
- Passos: criar sessão sem viagem, vincular, desvincular e vincular à outra moto.
- Resultado esperado: os três primeiros são aceitos e o último recusado.

**T3.10** — Versões e distâncias

- Pré-condição: sessão importada.
- Passos: calcular duas vezes com o mesmo método e uma com método novo.
- Resultado esperado: sem duplicata; a versão antiga permanece e não pode ser substituída; oficial, roda e Rideology aparecem separadas.

**T3.11** — GPS do CSV

- Pré-condição: fixture com trecho normal, congelado em movimento, salto e sem sinal, sem GPS do aplicativo.
- Passos: calcular a distância oficial.
- Resultado esperado: fonte `gps_csv`, trechos defeituosos fora, sem interpolação, cobertura mostrada e repetição com a moto parada não é congelamento.

**T3.12** — Prioridade do aplicativo

- Pré-condição: viagem com GPS do aplicativo e sessão CSV vinculada.
- Passos: calcular; repetir com o aplicativo sem par utilizável.
- Resultado esperado: primeiro `gps_app` sem completar lacunas com o CSV; depois `gps_csv`.

**T3.13** — Gráficos

- Pré-condição: sessão com lacuna de 203 s sintética e campos ausentes.
- Passos: abrir os gráficos.
- Resultado esperado: lacuna e ausência aparecem como interrupção, nunca como linha contínua ou zero.

**T3.14** — Leitura da fixture de 14 colunas

- Pré-condição: fixture do T1.18.
- Passos: importar.
- Resultado esperado: 14 colunas preservadas como lidas, constantes não viram ausência, N é `gear` N e a velocidade em m/s confere.

**T3.15** — Nova versão do parser

- Pré-condição: importação lida pela versão 1.
- Passos: ler com a versão 2.
- Resultado esperado: nova leitura e nova sessão; bruto, hash e sessão anterior iguais.

**T3.16** — Identidade só no bruto

- Pré-condição: moto com apelido digitado no aplicativo e fixture com outro nome e outro apelido sentinela no preâmbulo, no comentário e no nome do arquivo.
- Passos: importar, forçar erro, exportar, gerar entrada de relatório e buscar as sentinelas.
- Resultado esperado: aparecem só no arquivo bruto local, e o apelido digitado não muda.

**T3.17** — Exclusão de importação

- Pré-condição: duas importações com derivados.
- Passos: tentar alterar e apagar; autorizar sem a frase; autorizar corretamente a primeira; matar o processo no meio; concluir.
- Resultado esperado: só a autorizada some, com arquivo, derivados, registro permanente e contagem correta; dado novo no alvo é recusado e a outra fica intacta.

**T3.18** — Sessão sem fuso

- Pré-condição: fixture com título em minutos e sem fuso.
- Passos: importar; tentar gravar instante UTC sem offset.
- Resultado esperado: hora local com precisão de 60 s, instante UTC ausente com motivo; a gravação com UTC sem offset é recusada.

### F4

**T4.1** — Histórico 0, 1 e N

- Pré-condição: bases com 0, 1 e N viagens.
- Passos: comparar.
- Resultado esperado: 0 e 1 devolvem `insufficient_history` e N compara.

**T4.2** — Anomalia e ruído

- Pré-condição: séries sintéticas rotuladas.
- Passos: detectar.
- Resultado esperado: anomalia encontrada e taxa de falso positivo medida e registrada.

**T4.3** — Nenhuma afirmação de falha

- Pré-condição: achados de todos os tipos.
- Passos: inspecionar os textos.
- Resultado esperado: nenhum declara falha mecânica sem evidência suficiente.

**T4.4** — Troca de algoritmo

- Pré-condição: achados da versão 1.
- Passos: executar a versão 2.
- Resultado esperado: achados da versão 1 permanecem iguais.

**T4.5** — Sensor constante

- Pré-condição: série com valor constante.
- Passos: analisar.
- Resultado esperado: marcado como suspeita de indisponibilidade, sem virar medição normal nem ausência presumida.

**T4.6** — Tipos e evidência

- Pré-condição: métricas e linha de base.
- Passos: gerar um achado de cada tipo e um desvio sem linha de base.
- Resultado esperado: quatro tipos com evidência não vazia; o desvio sem linha de base é recusado.

### F5

**T5.1** — Citação inexistente

- Pré-condição: resposta simulada com citação falsa.
- Passos: validar.
- Resultado esperado: recusada com `invalid_citation`.

**T5.2** — Número fora da entrada

- Pré-condição: resposta simulada com número inventado.
- Passos: validar.
- Resultado esperado: recusada com `number_not_in_input`.

**T5.3** — Injeção em nota

- Pré-condição: nota e comentário com instruções maliciosas.
- Passos: gerar relatório.
- Resultado esperado: instruções são tratadas como dado e não mudam o resultado.

**T5.4** — Sem rede

- Pré-condição: modo avião.
- Passos: pedir relatório.
- Resultado esperado: números e evidências aparecem e a narrativa indica indisponibilidade, sem erro fatal.

**T5.5** — Conteúdo enviado

- Pré-condição: captura do envio, sem consentimento.
- Passos: gerar relatório.
- Resultado esperado: nenhuma coordenada ou identidade no envio.

**T5.6** — Relatório de viagem

- Pré-condição: viagem com métricas.
- Passos: gerar.
- Resultado esperado: versão, hash da entrada, evidências e incerteza presentes.

**T5.7** — Semanal e mensal

- Pré-condição: viagens na virada de semana, de mês e de fuso.
- Passos: gerar os dois.
- Resultado esperado: cada viagem entra uma vez no período correto, com offset registrado.

**T5.8** — Perguntas

- Pré-condição: histórico conhecido.
- Passos: fazer uma pergunta respondível e outra sem dados.
- Resultado esperado: a primeira cita as fontes e a segunda declara falta de dados.

### F6

**T6.1** — Registro HCI

- Pré-condição: moto e captura do tráfego Bluetooth.
- Passos: usar o adaptador.
- Resultado esperado: nenhuma operação fora da lista permitida.

**T6.2** — Porta sem escrita

- Pré-condição: código do adaptador.
- Passos: inspecionar a interface.
- Resultado esperado: não existe método de escrita para ECU, ABS, KTRC ou painel.

**T6.3** — Falhas de conexão

- Pré-condição: adaptador e moto.
- Passos: testar moto desligada, desconexão e aplicativo oficial conectado.
- Resultado esperado: erro tipado, dados já recebidos preservados e nenhum travamento.

**T6.4** — Alternativa por CSV

- Pré-condição: Bluetooth incompatível.
- Passos: usar o fluxo de importação.
- Resultado esperado: todas as funções da F3 seguem disponíveis.

### F7

**T7.1** — Backup e restauração

- Pré-condição: dados completos.
- Passos: fazer backup, desinstalar, reinstalar e restaurar.
- Resultado esperado: contagens e hashes iguais.

**T7.2** — Exportação reimportável

- Pré-condição: dados completos.
- Passos: exportar e importar em instalação limpa.
- Resultado esperado: contagens e hashes dos dados exportados iguais; executar na fase em que RF-15 for alocado.

**T7.3** — Rastros após exclusão

- Pré-condição: dados excluídos definitivamente.
- Passos: inspecionar banco, arquivos, cache e logs.
- Resultado esperado: não resta conteúdo excluído; permanece só o registro da exclusão.

**T7.4** — Logs

- Pré-condição: uso completo com coordenadas e identidades sentinela.
- Passos: inspecionar os logs.
- Resultado esperado: nenhuma sentinela encontrada.

**T7.5** — Conteúdo da exportação

- Pré-condição: importação com nome e apelido sentinela.
- Passos: exportar e inspecionar.
- Resultado esperado: sentinelas ausentes e bruto CSV com identidade não incluído; executar na fase em que RF-15 for alocado.

## T0.5 — confronto executado

Fonte: relatório de estrutura de **um único arquivo** real, mantido fora do Git. O CSV não foi aberto para este confronto, e nada aqui vale para outros arquivos sem nova verificação. Toda célula das linhas de dados fica como lida em `telemetry_sample.raw_values_json`; a coluna do modelo abaixo é a normalização.

| Coluna do CSV | Coluna do modelo | Resultado |
|---|---|---|
| `elapsed_msec` | `elapsed_ms` | representada |
| `gps_latitude` | `latitude` | representada; vírgula decimal entre aspas |
| `gps_longitude` | `longitude` | representada; vírgula decimal entre aspas |
| `instant_fuel_consumption(km/L)` | `fuel_km_per_l` | representada; 0.0 e 99.9 preservados, significado [A VERIFICAR] |
| `water_temperature(℃)` | `engine_temp_c` | representada; mesma unidade |
| `boost_temperature(℃)` | `raw_values_json` | preservada como lida; constante 0.0 |
| `engine_RPM` | `rpm` | representada |
| `wheel_speed(km/h)` | `speed_mps` | conversão por 3,6; valor em km/h em raw_values_json |
| `acceleration(G)` | `raw_values_json` | preservada como lida; constante -2.0 |
| `throttle_position(%)` | `raw_values_json` | preservada como lida; constante 0 |
| `boost_pressure(%)` | `raw_values_json` | preservada como lida; constante 0 |
| `gear_position` | `gear` | representada, inclusive N |
| `lean_angle(°)` | `raw_values_json` | preservada como lida; constante 0 |
| `rideology_score` | `raw_values_json` | preservada como lida |

Constantes não são tratadas como medição confirmada nem como ausência presumida.

| Metadado ou fato | Onde fica | Resultado |
|---|---|---|
| UserAccount Name | só no arquivo bruto | fora do banco por D3 |
| Vehicle Nickname | só no arquivo bruto | fora do banco por D3; `motorcycle.nickname` é digitado e independente |
| Title | `metadata_json.title` e `started_local` | representado; sem fuso, precisão de 60 s |
| Comment | `metadata_json.comment` | representado; omitido se contiver identidade |
| Distance(km) | `metadata_json.distance_km` e `distance_rideology` | representado, comparativo |
| Time(min) | `metadata_json.time_min` | representado, comparativo |
| R.Score | `metadata_json.rideology_score` | representado |
| Encoding CP932 e vírgula como delimitador | `telemetry_parse_run` | representado por leitura |
| Decimal diferente por coluna | `decimal_rules_json` | representado |
| Sem instante absoluto por linha | `recorded_at` ausente com motivo | representado |
| Lacuna de 203 s | diferença entre `elapsed_ms` consecutivos | calculável, não interpolada; nenhuma linha é inventada para o intervalo |
| Coordenadas repetidas, sem precisão | `gps_evaluation` | regra própria para o CSV |

Resultado: T0.5 passa na estrutura. A fixture sintética exigida pelo ADR 0001 **ainda não existe**: está atribuída à F1 (T1.18) e o critério não está cumprido. Não foram verificados os valores nem a existência de outros formatos.

## Execução de T0.1 e T0.3

A conferência de T0.1 extrai os IDs de `requirements.md`, compara cada linha com a matriz, exige definição única de cada teste e exige que todo teste definido apareça na matriz. A de T0.3 extrai os blocos `sql` sem alteração, em banco temporário.

## Correspondência com a análise independente

Todos os 45 testes originais mantêm o ID. Um deles não mantém o significado por inteiro.

| Original | Enunciado original | ID final | Título final | Mesmo significado |
|---|---|---|---|---|
| T0.1 | cada requisito tem ID, fase e teste | T0.1 | Rastreabilidade | sim |
| T0.2 | cada decisão tem dono e reversibilidade | T0.2 | Decisões | sim |
| T0.3 | um implementador cria o esquema sem perguntar | T0.3 | Esquema sem perguntas | sim |
| T0.4 | cada item da l.25 existe | T0.4 | Entregáveis | sim |
| T0.5 | confrontar o modelo com o CSV real | T0.5 | Confronto com a estrutura do CSV | não: o original confronta com o CSV real; o final usa o relatório de estrutura, porque o arquivo não é aberto nem versionado |
| T0.6 | varrer segredos/dados pessoais no histórico | T0.6 | Dados reais e segredos | sim |
| T1.1 | salvar e matar o processo em seguida | T1.1 | Morte após salvar | sim |
| T1.2 | instalação limpa em modo avião com reboot | T1.2 | Instalação limpa sem rede | sim |
| T1.3 | matar durante migração | T1.3 | Morte durante migração | sim |
| T1.4 | disco cheio | T1.4 | Disco cheio | sim |
| T1.5 | migrar de cada versão anterior | T1.5 | Migração de cada versão | sim |
| T1.6 | odômetro regressivo | T1.6 | Odômetro regressivo | sim |
| T1.7 | demonstração não contamina dados reais | T1.7 | Demonstração não contamina | sim |
| T2.1 | kill -9 em cada estado e medir pontos perdidos | T2.1 | Morte em cada estado | sim; acrescenta limite [PROPOSTA] |
| T2.2 | reboot na viagem | T2.2 | Reinício na viagem | sim |
| T2.3 | revogar permissão | T2.3 | Permissão revogada | sim |
| T2.4 | 2 h com tela apagada em modo economia, medindo lacunas e bateria | T2.4 | Tela apagada em economia | sim |
| T2.5 | injetar salto de 10 km | T2.5 | Salto de 10 km | sim |
| T2.6 | pausar/retomar 20x sem duplicar | T2.6 | Pausar e retomar 20 vezes | sim |
| T2.7 | mudar o relógio +-1 h | T2.7 | Relógio alterado | sim |
| T3.1 | sha256 do original igual antes/depois | T3.1 | Hash do original | sim |
| T3.2 | reimportar e renomear: zero duplicatas | T3.2 | Reimportar e renomear | sim |
| T3.3 | matriz encoding x delimitador x decimal | T3.3 | Matriz de formatos | sim |
| T3.4 | remover cada coluna: métrica fica ausente, não 0 | T3.4 | Coluna removida | sim |
| T3.5 | matar no meio: nada parcial visível | T3.5 | Morte durante importação | sim |
| T3.6 | arquivo grande sem travar a UI | T3.6 | Arquivo grande | sim; acrescenta limite [PROPOSTA] |
| T3.7 | conferir métricas contra cálculo independente com tolerância definida | T3.7 | Cálculo independente | sim; tolerância [PROPOSTA] |
| T4.1 | histórico com 0/1/N viagens | T4.1 | Histórico 0, 1 e N | sim |
| T4.2 | anomalia sintética e ruído (medir falsos positivos) | T4.2 | Anomalia e ruído | sim |
| T4.3 | nenhum texto afirma falha | T4.3 | Nenhuma afirmação de falha | sim |
| T4.4 | trocar o algoritmo preserva o resultado versionado | T4.4 | Troca de algoritmo | sim |
| T4.5 | sensor preso em valor constante | T4.5 | Sensor constante | sim |
| T5.1 | citação inexistente rejeitada | T5.1 | Citação inexistente | sim |
| T5.2 | número fora da entrada rejeitado | T5.2 | Número fora da entrada | sim |
| T5.3 | injeção em nota | T5.3 | Injeção em nota | sim |
| T5.4 | offline degrada sem erro | T5.4 | Sem rede | sim |
| T5.5 | capturar o payload e provar ausência de coordenadas sem consentimento | T5.5 | Conteúdo enviado | sim |
| T6.1 | registro HCI sem operação fora da lista permitida | T6.1 | Registro HCI | sim |
| T6.2 | porta sem método de escrita | T6.2 | Porta sem escrita | sim |
| T6.3 | moto desligada, desconexão e app oficial conectado | T6.3 | Falhas de conexão | sim |
| T6.4 | fallback CSV | T6.4 | Alternativa por CSV | sim |
| T7.1 | backup -> desinstalar -> restaurar com contagem e hash iguais | T7.1 | Backup e restauração | sim |
| T7.2 | exportação reimportável | T7.2 | Exportação reimportável | sim |
| T7.3 | excluir dados remove rastros locais | T7.3 | Rastros após exclusão | sim |
| T7.4 | logs sem coordenadas | T7.4 | Logs | sim |
