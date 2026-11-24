# Arquitetura

## Camadas

| Camada | Contém | Pode depender de |
|---|---|---|
| `domain` | entidades, invariantes, estados e cálculos puros, em TypeScript | nada |
| `application` | casos de uso e portas | `domain` |
| `adapters` | implementações das portas: SQLite, arquivos, GPS, importador, relatórios | `application`, `domain` |
| `ui` | telas React Native e navegação | `application` |

A UI não acessa SQLite, arquivos nem GPS diretamente. Os tipos das portas são de domínio; SQL e tipos de biblioteca não atravessam a fronteira.

## Módulos

| Módulo | Responsabilidade | Não faz |
|---|---|---|
| `garage` | moto, odômetro, dados de demonstração | manutenção |
| `maintenance` | registros, itens, planos e próximos vencimentos | ler telemetria |
| `trips` | estados da viagem, segmentos, gravação e recuperação do GPS | calcular métricas |
| `telemetry` | arquivo bruto, leituras do parser, sessões, amostras, problemas e vínculo com viagem | interpretar anomalia |
| `analytics` | validade do GPS, métricas, linha de base e achados, todos versionados | alterar bruto |
| `reports` | entrada estruturada, narrativa, evidência e incerteza | calcular número novo |
| `privacy` | exclusão definitiva autorizada e exportação manual sem identidade | apagar por prazo |

## Fluxo de dados

```text
 React Native UI
       |
       v
 casos de uso (application) ----> Clock
       |
       +--> TripPort / TelemetryPort / MaintenancePort --> SQLite (fonte local de verdade)
       |
       +--> LocationPort <-- GPS do aparelho
       |        `--> location_point (bruto)
       |
       +--> ImportPort <-- arquivo CSV escolhido pela pessoa
       |        |--> cópia bruta local + SHA-256 --> telemetry_import
       |        `--> parser versionado --> telemetry_sample + sample_issue
       |
       +--> MetricPort: bruto --> gps_evaluation --> metric_result --> baseline/finding
       |
       `--> ReportPort: métricas + evidências --> report --> UI
```

O GPS do CSV entra em `telemetry_sample` e o do aplicativo em `location_point`. Eles só se encontram na avaliação e na métrica.

## Contratos das portas

Toda operação devolve resultado ou erro tipado, sem exceção como fluxo normal. Operação que altera estado é uma transação: ou tudo é gravado, ou nada. Erros comuns a todas as portas de dados: `storage_full`, `storage_corrupt`, `constraint_violation`, `schema_too_new`.

### MaintenancePort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `saveMotorcycle` | dados da moto | moto gravada | `duplicate_vin` |
| `addOdometer` | moto, valor em metros, instante, offset, origem | leitura | `odometer_regression` |
| `correctOdometer` | leitura corrigida, novo valor, nota | nova leitura | `odometer_regression`, `already_corrected`, `note_required` |
| `saveRecord` | registro, itens e planos cumpridos | registro com itens | `money_incomplete`, `motorcycle_mismatch` |
| `savePlan` | intervalos e itens previstos | plano | `interval_required` |
| `listHistory` | moto e intervalo | registros ordenados | — |

### TripPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `create` | moto e origem | viagem `draft` | `demo_mismatch` |
| `transition` | viagem e evento da tabela de estados | viagem e segmento aberto, se houver | `invalid_transition`, `trip_already_active` |
| `appendPoints` | segmento e lote ordenado de pontos | quantidade gravada | `duplicate_sequence`, `segment_closed` |
| `findRecoverable` | moto | viagem aberta encontrada, ou nada | — |

### TelemetryPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `registerImport` | moto, hash, tamanho, nome tratado | importação nova ou já existente | `hash_owned_by_other_motorcycle` |
| `saveParse` | importação, versão do parser, sessão, amostras, problemas | leitura final | `already_parsed`, `identity_not_allowed` |
| `linkTrip` | sessão e viagem, ou nenhuma | sessão | `motorcycle_mismatch` |
| `readSamples` | sessão e intervalo de `elapsed_ms` | amostras ordenadas | — |

### ImportPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `copyRaw` | referência do arquivo escolhido | hash, tamanho e caminho bruto | `file_unreadable`, `copy_hash_mismatch` |
| `detect` | arquivo bruto | encoding, delimitador, regra decimal por coluna, preâmbulo e cabeçalho | `unknown_format`, `ambiguous_encoding` |
| `parse` | arquivo, detecção, versão do parser, sinal de cancelamento | metadados permitidos, amostras, problemas | `empty_file`, `header_only`, `missing_elapsed_column`, `cancelled` |

### LocationPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `permission` | — | concedida precisa, aproximada, negada | — |
| `start` | política de amostragem | fluxo de pontos e eventos | `permission_denied`, `location_disabled`, `service_not_allowed` |
| `stop` | — | confirmação | — |

Eventos do fluxo: `point`, `permission_revoked`, `location_disabled`, `signal_lost`.

### MetricPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `evaluateGps` | viagem ou sessão e versão da regra | avaliações | `unknown_rule_version` |
| `calculate` | escopo, tipo, versão do método | métrica com valor ou motivo, cobertura e evidência | `unknown_method_version` |
| `compareWithBaseline` | métrica e versão da linha de base | achados | `insufficient_history` |

Recalcular a mesma entrada com o mesmo método devolve o resultado existente.

### ReportPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `buildInput` | escopo e período | entrada estruturada e hash, sem identidade e sem coordenada | `no_data` |
| `generate` | entrada estruturada | narrativa, citações e incerteza | `offline`, `timeout`, `invalid_citation`, `number_not_in_input` |

### PrivacyPort

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `previewDeletion` | importação ou viagem | contagem do que será apagado | `target_not_found`, `trip_still_active` |
| `deleteRaw` | alvo, contagem mostrada, motivo e frase de confirmação | registro de exclusão concluído | `not_confirmed`, `count_changed`, `file_removal_failed` |
| `exportData` | escopo | arquivo reimportável e relação de hashes | `destination_unwritable` |

### Clock

| Operação | Entrada | Saída | Erros próprios |
|---|---|---|---|
| `now` | — | instante UTC em milissegundos e offset local | — |
| `monotonic` | — | milissegundos monotônicos | — |

## Máquina de estados da viagem

| Estado | Evento | Guarda | Próximo | Efeito |
|---|---|---|---|---|
| — | `create` | moto existe | `draft` | grava viagem sem segmento |
| `draft` | `start` | permissão precisa; nenhuma viagem ativa na moto | `recording` | abre segmento 0 e inicia GPS |
| `recording` | `pause` | — | `paused` | grava lote, fecha segmento, para GPS |
| `paused` | `resume` | permissão precisa | `recording` | abre novo segmento |
| `recording` | `permission_revoked` | — | `interrupted` | grava lote, marca segmento `interrupted`, avisa |
| `paused` | `permission_revoked` | — | `paused` | avisa; `resume` fica bloqueado pela guarda |
| `recording` | `process_killed`, detectado na abertura | segmento aberto | `interrupted` | segmento `interrupted`, com fim no último ponto gravado ou no início, se não houver ponto |
| `paused` | `process_killed`, detectado na abertura | nenhum segmento aberto | `paused` | nada a recuperar |
| `interrupted` | `recover` | confirmação; permissão precisa | `recording` | abre novo segmento |
| `interrupted` | `keep_paused` | confirmação | `paused` | não abre segmento |
| `interrupted` | `finish` | confirmação | `finished` | fim no último ponto gravado |
| `recording`, `paused` | `finish` | confirmação | `finished` | grava lote e fecha segmento, se houver |
| `draft` | `cancel` | confirmação | `cancelled` | não apaga nada |
| `recording`, `paused`, `interrupted` | `cancel` | confirmação | `cancelled` | fecha segmento; pontos permanecem |

`finished` e `cancelled` são finais. Evento repetido, como dois toques em iniciar, devolve `invalid_transition` sem efeito. Entre segmentos nunca há distância.

## Gravação do GPS

Cada lote fecha ao completar 5 s ou 20 pontos, o que vier primeiro, em uma transação. A perda máxima proposta em morte do processo é 5 s de pontos [PROPOSTA], a confirmar no spike S1. Ponto inválido também é gravado; sua exclusão da distância ocorre na avaliação.

A amostragem é adaptativa: 1 s em movimento e 5 s parado [PROPOSTA]. Parada é velocidade abaixo de 1 m/s por 60 s ou mais [PROPOSTA], registrada como métrica derivada.

## Importação

1. Copiar os bytes para a área bruta e conferir o SHA-256 da cópia.
2. Detectar, por arquivo, encoding e delimitador, e por coluna o separador decimal. O arquivo observado é CP932, com vírgula como delimitador, vírgula decimal entre aspas em latitude e longitude e ponto nas demais colunas.
3. Ler o preâmbulo. No arquivo observado são sete linhas de metadados e o cabeçalho na oitava; outros arquivos podem diferir e não são rejeitados só por isso.
4. Guardar o texto de cada célula e normalizar as colunas conhecidas. Colunas desconhecidas seguem em `raw_values_json`, exceto as de identidade; o que não puder ser interpretado vira ausente com motivo.
5. Omitir nome de usuário e apelido do veículo em tudo que sai do arquivo bruto.
6. Gravar sessão, amostras e problemas em uma transação. Cancelamento ou falha não deixa nada parcial visível.

## Offline

Funcionam sem rede: cadastro, manutenção, odômetro, demonstração, viagem e GPS, importação, métricas, linha de base, consultas, exportação manual e exclusão. Só a narrativa gerada por provedor remoto (F5) pode exigir rede; sem ela, os números e evidências continuam disponíveis.

## Spikes antes da F1

Documentados aqui e não executados na F0. As metas são [PROPOSTA].

### S1 — GPS com tela apagada

- **Pergunta:** o Galaxy S25 Ultra de referência mantém o fluxo em primeiro plano com a tela apagada?
- **Método:** uma sessão de 2 h com economia de bateria desligada e outra com ela ligada; medir pontos esperados e recebidos, lacunas, bateria e comportamento ao revogar a permissão e ao matar o processo.
- **Sucesso:** ao menos 98% dos pontos esperados; nenhuma lacuna acima de 10 s com céu aberto; perda de até 5 s ao matar o processo; consumo de até 10 pontos percentuais em 2 h.
- **Limite:** dois dias.

### S2 — Estrutura do CSV com fixture sintética

- **Pergunta:** o parser aceita a estrutura conhecida e recusa as demais com motivo?
- **Método:** matriz de encoding (CP932, UTF-8 com e sem BOM, UTF-16, Windows-1252), delimitador (`,`, `;`, tabulação) e decimal (por coluna), mais preâmbulo ausente e coluna removida, só com dados sintéticos.
- **Sucesso:** 100% dos casos aceitos com valores iguais aos esperados ou recusados com código de motivo; zero coluna descartada em silêncio.
- **Limite:** um dia.

### S3 — SQLite no aparelho

- **Pergunta:** a biblioteca candidata oferece o que o esquema usa, com latência aceitável?
- **Método:** executar os blocos de `data-model.md`, gravar 72.000 pontos em lotes de 20 e importar 6.000 amostras sintéticas.
- **Sucesso:** versão 3.37 ou superior com `STRICT`, JSON, WAL e pragmas por conexão; p95 por lote até 250 ms; importação até 10 s; `integrity_check` igual a `ok` após matar o processo durante a gravação.
- **Limite:** um dia.

## Estrutura do repositório

Na F0 existem apenas marcadores de diretório em `src/domain`, `src/application`, `src/adapters`, `src/ui` e `tests`. Na F1 cada módulo é uma subpasta de sua camada, e os testes ficam em `tests/unit`, `tests/integration` e `tests/fixtures` (só sintéticas).

## Estratégia de testes

| Nível | Cobre |
|---|---|
| Unidade | domínio: estados, validade do GPS, distância, métricas, dinheiro |
| Contrato | cada porta, com a mesma bateria para implementação real e falsa |
| Integração | esquema, gatilhos, migrações a partir de cada versão, importação com fixtures |
| Aparelho | GPS, ciclo de vida, bateria, permissões, modo avião |

Migração verifica backup antes, roda em transação e executa `integrity_check` depois. Nenhuma porta oferece escrita em ECU, ABS, KTRC ou painel.
