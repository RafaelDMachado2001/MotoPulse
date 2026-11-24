# Requisitos

A coluna Teste coincide com a matriz de `acceptance-criteria.md`.

## Funcionais

| ID | Descrição verificável | Fase | Teste |
|---|---|---|---|
| RF-01 | Criar e editar uma motocicleta, com apelido opcional digitado no aplicativo, e reencontrá-la após reabrir o aplicativo | F1 | T1.8 |
| RF-02 | Registrar manutenção, com custo e moeda juntos ou com motivo de ausência | F1 | T1.9 |
| RF-03 | Registrar itens ligados a um registro de manutenção | F1 | T1.10 |
| RF-04 | Criar plano por distância e/ou tempo e vincular sua execução a um registro | F1 | T1.11 |
| RF-05 | Registrar odômetro sem regressão e corrigir leitura de forma rastreável | F1 | T1.6, T1.12 |
| RF-06 | Ler dados de demonstração sintéticos sem rede e separados dos reais | F1 | T1.7, T1.13 |
| RF-07 | Criar, pausar, retomar e encerrar viagem conforme a tabela de estados | F2 | T2.6, T2.8 |
| RF-08 | Recuperar viagem interrompida por morte do processo, reinício ou permissão revogada | F2 | T2.1, T2.2, T2.3 |
| RF-09 | Calcular distância oficial só com GPS válido, sem interpolar, mostrando a cobertura | F2 | T2.5, T2.7, T2.10 |
| RF-10 | Importar arquivo do Rideology preservando bytes, hash, origem e problemas encontrados | F3 | T3.1, T3.2, T3.3, T3.8 |
| RF-11 | Associar viagem e sessão de telemetria, com vínculo opcional dos dois lados | F3 | T3.9 |
| RF-12 | Calcular métricas versionadas, com distâncias oficial, da roda e do Rideology separadas | F3 | T3.4, T3.7, T3.10 |
| RF-13 | Emitir achados que distinguem medição, desvio, hipótese e recomendação, com evidência | F4 | T4.1, T4.2, T4.3, T4.4, T4.5, T4.6 |
| RF-14 | Gerar relatório de viagem com versão, evidência e incerteza | F5 | T5.1, T5.2, T5.6 |
| RF-15 | Exportar dados manualmente em formato reimportável; fase aberta, recomendação F1 | F1 ou F7, aberta | T7.2, T7.5 |
| RF-16 | Ler fixture sintética com as 14 colunas do ADR 0001, preservando constantes e marcha N | F3 | T3.14 |
| RF-17 | Navegar entre garagem, manutenção, viagens e relatórios | F1 | T1.14 |
| RF-18 | Amostrar o GPS de forma adaptativa | F2 | T2.11 |
| RF-19 | Classificar paradas | F2 | T2.12 |
| RF-20 | Exibir gráficos das métricas importadas, com lacunas visíveis | F3 | T3.13 |
| RF-21 | Gerar relatórios semanal e mensal | F5 | T5.7 |
| RF-22 | Responder perguntas sobre o histórico com base em dados citados | F5 | T5.8 |
| RF-23 | Usar o GPS do CSV na distância oficial, excluindo congelado, salto e sem sinal | F3 | T3.11 |
| RF-24 | Dar prioridade ao GPS do aplicativo quando ele e o CSV existirem | F3 | T3.12 |
| RF-25 | Excluir bruto definitivamente só com confirmação explícita e registro rastreável | F3 (viagem: F2) | T2.14, T3.17 |
| RF-26 | Ler a moto por Bluetooth, somente leitura, com alternativa por CSV | F6 | T6.3, T6.4 |
| RF-27 | Importar sessão sem fuso conhecido, sem inventar instante UTC | F3 | T3.18 |

## Não funcionais

| ID | Descrição verificável | Fase | Teste |
|---|---|---|---|
| RNF-01 | SQLite local é a fonte de verdade e nenhuma transação expõe estado parcial | F1 | T1.4, T1.17, T3.5 |
| RNF-02 | UUID, UTC com offset, unidades canônicas e ausência com motivo | F0 e F1 | T0.3, T1.16 |
| RNF-03 | Bruto imutável e derivado recalculável e versionado, sem substituição | F3 | T3.1, T3.15 |
| RNF-04 | Dados reais não entram no Git; fixtures sintéticas entram | F0 | T0.6, T1.18 |
| RNF-05 | Nenhuma operação de escrita em ECU, ABS, KTRC ou painel | F6 | T6.1, T6.2 |
| RNF-06 | Registro confirmado sobrevive à morte do processo e à reabertura | F1 | T1.1 |
| RNF-07 | Capacidades locais funcionam sem rede | F1 | T1.2, T1.15 |
| RNF-08 | Exclusão e alteração de bruto são barradas por padrão | F2 e F3 | T2.14, T3.17 |
| RNF-09 | Nome de usuário e apelido do veículo do CSV não saem do arquivo bruto; logs não têm coordenadas | F3 | T3.16, T7.4 |
| RNF-10 | Migração é transacional e parte de qualquer versão anterior | F1 | T1.3, T1.5 |
| RNF-11 | Importar arquivo grande não trava a interface | F3 | T3.6 |
| RNF-12 | Gravação com tela apagada respeita as metas de lacuna e bateria | F2 | T2.4 |
| RNF-13 | IA resiste a conteúdo malicioso, degrada sem rede e não recebe coordenada sem consentimento | F5 | T5.3, T5.4, T5.5 |
| RNF-14 | Backup posterior restaura contagens e hashes; exclusão remove rastros locais | F7 | T7.1, T7.3 |
| RNF-15 | Contrato da F0 é completo, rastreável e confrontado com a estrutura do CSV | F0 | T0.1, T0.2, T0.4, T0.5 |

## Funciona sem rede

Demonstração, moto, manutenção, odômetro, viagens e GPS, importação, métricas, gráficos, linha de base, achados, consulta, exportação manual e exclusão definitiva. Na F5, apenas a narrativa de provedor remoto pode exigir rede.

## Fora de escopo

- Do MVP: backend, sincronização e qualquer escrita na moto.
- De F0 e F1: Bluetooth, IA remota e backup criptografado, que o ADR 0001 deixa para fase posterior.
- Exportação manual **não** está fora de escopo: só a fase está aberta.

## Lacunas abertas

- Decisões do usuário pendentes: ver a tabela do ADR 0002.
- [A VERIFICAR]: significado de 0.0 e 99.9 km/L; relação entre o título do CSV e `elapsed` zero; representação de perda de sinal no CSV; estrutura de outros CSVs; causa da lacuna observada; limiares de GPS e de anomalia; troca de painel do odômetro; versão do SQLite embutido.
