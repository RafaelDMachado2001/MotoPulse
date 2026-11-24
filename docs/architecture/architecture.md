# Arquitetura

## Princípios

MotoPulse é Android offline-first para a Kawasaki Ninja ZX-4R 2026. O SQLite local é a fonte de verdade; sincronização, backup criptografado e BLE permanecem posteriores. Dados brutos são imutáveis, derivados são recalculáveis e versionados.

## Camadas e portas

- `domain`: entidades, value objects, invariantes, máquina de estados e cálculo determinístico; não conhece React Native nem SQLite.
- `application`: casos de uso transacionais e portas (`TripRepository`, `TelemetryRepository`, `LocationPort`, `ImportPort`, `ReportPort`, `Clock`).
- `adapters`: SQLite/migrations, filesystem, GPS, importador Rideology, relatórios e futuros BLE/IA; implementa portas, sem lógica de apresentação.
- `ui`: React Native/TypeScript, navegação, permissões e estados de tela; nunca acessa banco diretamente.

Fluxo: UI → caso de uso → porta → adaptador SQLite/GPS/arquivo; eventos persistidos retornam à UI. Importação é `bytes + SHA-256 → cópia bruta imutável → detecção/parsing → normalização em transação → issues e métricas`; nenhuma coluna desconhecida é inventada. Relatórios consomem métricas estruturadas e evidências.

## Viagem

`draft → recording → paused ↔ recording → finished`; interrupção inesperada na abertura vira `interrupted`, podendo ser recuperada para `recording` ou encerrada. Cada segmento registra início/fim e pontos com sequência. Distância oficial usa apenas GPS válido, sem interpolar lacunas; roda e Rideology são comparativas (ADR 0001). Associação com sessão Rideology é opcional nos dois lados.

## Repositório e testes

`src/{domain,application,adapters,ui}`, `tests/` (unitários, integração SQLite/importação, contrato e cenários de ciclo de vida), `docs/architecture` e `docs/decisions`. Migrations usam transação, `foreign_keys=ON`, WAL e `integrity_check`; fixtures são sintéticas/anonimizadas e CSV real nunca entra no Git.

## Spikes antes da F1

Documentar (não executar nesta fase): GPS com tela apagada no Galaxy S25 Ultra; estrutura/encoding/unidades do CSV com fixture sintética. Não há escrita em ECU, ABS, KTRC ou painel.
