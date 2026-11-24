# Arquitetura

## Camadas, módulos e portas

`domain` contém entidades, invariantes, estados e cálculos puros; `application` orquestra casos de uso; `adapters` implementa SQLite/migrations, filesystem, GPS, Rideology e relatórios; `ui` é React Native/TypeScript e só chama casos de uso. Módulos: garage, maintenance, trips, telemetry, analytics e reports. A UI não acessa SQLite diretamente.

Portas: `TripRepository` (criar, transicionar, lotear pontos, recuperar); `TelemetryRepository` (imports/sessões/amostras/issues); `MaintenanceRepository` (registros/itens/planos/odômetro); `LocationPort` (permissão, iniciar/parar, stream); `ImportPort` (bytes, hash, parse e issues); `MetricPort` (calcular/versionar); `ReportPort` (estrutura/evidência/incerteza); `Clock` (UTC e monotônico). Cada porta recebe tipos de domínio, retorna resultado/erro explícito, é transacional quando altera estado e não expõe SQL à UI.

## Fluxo de dados

```text
React Native UI -> casos application -> portas
      |                                |-> SQLite (fonte local)
      |                                |-> GPS -> location_point -> métricas
arquivo Rideology -> hash/cópia bruta -> parser -> issues/samples -> métricas
      métricas + evidências -> relatório -> UI
```

Bruto é imutável; derivados são recalculáveis/versionados. Offline cobre cadastro, manutenção, viagem, GPS, importação local, métricas e relatórios locais.

## Máquina de viagem

| Estado | Evento | Guarda | Efeito |
|---|---|---|---|
| draft | start | moto válida/permissão | cria segmento, inicia gravação |
| recording | pause | viagem aberta | fecha segmento, paused |
| paused | resume | permissão disponível | novo segmento, recording |
| recording/paused | process_killed | detectado na abertura | interrupted, preserva pontos |
| interrupted | recover | confirmação | novo segmento, recording |
| interrupted | finish | confirmação | finished, fecha tempos |
| recording/paused | finish | viagem aberta | finished, persiste resumo |
| qualquer não final | cancel | confirmação | cancelled, não apaga bruto |

GPS é gravado em lotes de até 5 s ou 20 pontos (primeiro limite), com transação por lote; perda máxima proposta em morte do processo é 5 s, a verificar no spike. Pontos inválidos persistem para auditoria mas não entram na distância oficial.

## Spikes pré-F1 (documentar, não executar nesta fase)

| Spike | Pergunta | Método | Sucesso | Limite |
|---|---|---|---|---|
| GPS tela apagada | Galaxy S25 Ultra mantém stream? | foreground/background, permissões, economia, bateria e lacunas | pontos dentro da meta a definir sem morte do serviço | 1 sessão, 2 h |
| CSV sintético | quais encoding/delimitador/decimal e estrutura? | matriz de fixtures sintéticas; colunas reais ficam [A VERIFICAR] | parser identifica formato e marca desconhecidos | 1 dia |
| SQLite volume | lotes preservam latência? | fixture sintética, transação e integrity_check | sem estado parcial e UI responsiva | 1 dia |

## Testes e estrutura

`src/{domain,application,adapters,ui}`, `tests/{unit,integration,fixtures}` (fixtures sintéticas permitidas), docs. Testar invariantes, migrations transacionais, ciclo de vida, hash, ausência/unidades e contratos de portas. SQLite usa `foreign_keys=ON`, WAL e `integrity_check`; não há BLE nem escrita em ECU/ABS/KTRC/painel.
