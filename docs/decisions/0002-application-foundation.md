# ADR 0002 — Fundação da aplicação

## Status

Proposto para revisão, exceto as decisões D1, D2 e D3, aceitas pelo usuário em 2026-11-24. Em conflito, o ADR 0001 prevalece.

## Contexto

O MotoPulse precisa de uma base Android que funcione sem rede, preserve dados brutos e permita recalcular resultados. A estrutura de um único CSV real do Rideology é conhecida por um relatório mantido fora do Git; o arquivo existe, mas não entra no repositório. Outros arquivos podem diferir.

O plano técnico e a análise independente da F0 divergiram em vários pontos, arbitrados abaixo. Este registro distingue o que foi decidido pelo usuário, arbitrado para a F0, proposto e deixado em aberto.

## Escala de reversibilidade

Usada em todas as decisões deste documento.

| Nível | Significado |
|---|---|
| fácil | muda sem migrar dados nem refazer trabalho relevante |
| moderada | exige configuração, adaptação de código ou migração simples |
| difícil | exige migrar dados reais ou refazer parte relevante do produto |
| **IRREVERSÍVEL** | não pode ser desfeita depois de acontecer |

## Decisões aceitas pelo usuário

### D1 — GPS congelado, com salto ou sem sinal

- **Decisão:** esses trechos ficam fora da distância oficial, sem interpolação e com a cobertura mostrada. Havendo GPS do aplicativo e do CSV, o do aplicativo tem prioridade.
- **Data:** 2026-11-24.
- **Alternativas:** interpolar; usar o GPS do CSV só como comparação; combinar as fontes.
- **Consequência:** a distância oficial pode ser menor que a percorrida, e a cobertura explica por quê. Sem GPS do aplicativo, o do CSV pode compor a distância oficial. A prioridade vale para a viagem inteira: o CSV não completa lacunas do aplicativo. Os limiares continuam [PROPOSTA] ou [A VERIFICAR].
- **Reversibilidade:** moderada; a validade é derivada e recalculável.
- **Dono:** usuário.

### D2 — Exclusão de bruto

- **Decisão:** bloqueada por padrão; definitiva só com confirmação explícita.
- **Data:** 2026-11-24.
- **Alternativas:** exclusão em cascata; nunca permitir exclusão.
- **Consequência:** gatilhos barram alteração e exclusão, e `raw_deletion` registra a autorização e o resultado de forma permanente. Com autorização aberta, o alvo não recebe dado novo. Derivados só podem ser apagados junto com o alvo; a linha de base, agregada, permanece. Retenção por prazo segue aberta.
- **Reversibilidade:** a política é moderada; cada exclusão executada é **IRREVERSÍVEL**.
- **Dono:** usuário.

### D3 — Nome de usuário e apelido do veículo do CSV

- **Decisão:** ficam só no arquivo bruto local; fora de banco, inclusive `metadata_json`, de logs, de exportação e de IA.
- **Data:** 2026-11-24.
- **Alternativas:** guardar no banco e ocultar na interface; remover também do bruto, o que quebraria o hash.
- **Consequência:** o banco aceita apenas metadados de uma lista permitida. O cadastro da moto tem apelido opcional, digitado no aplicativo, que nunca é preenchido a partir do CSV. O arquivo bruto não é exportado com identidade.
- **Reversibilidade:** a regra é fácil de mudar; um vazamento é **IRREVERSÍVEL**.
- **Dono:** usuário.

## Decisões arbitradas para a F0

### A1 — Camadas e portas

- **Decisão:** `domain`, `application`, `adapters` e `ui`.
- **Alternativas:** organizar só por funcionalidade; um único módulo.
- **Consequência:** domínio testável sem aparelho, com mais arquivos de interface.
- **Reversibilidade:** fácil antes da F1; moderada depois.
- **Dono:** equipe técnica.

### A2 — Identificadores, tempo, unidades e ausência

- **Decisão:** UUID, UTC com offset original, unidades canônicas e NULL com motivo.
- **Alternativas:** inteiros sequenciais, hora local, zero como ausência.
- **Consequência:** esquema mais verboso, sem ambiguidade entre zero e ausência.
- **Reversibilidade:** difícil após dados reais.
- **Dono:** equipe técnica.

### A3 — Dinheiro em centavos com moeda

- **Decisão:** inteiro em centavos e código de três letras, sempre juntos.
- **Alternativas:** número decimal; moeda implícita.
- **Consequência:** sem erro de arredondamento. Moedas sem duas casas decimais exigiriam regra própria.
- **Reversibilidade:** difícil após dados reais.
- **Dono:** equipe técnica.

### A4 — Manutenção e odômetro

- **Decisão:** registros, itens e planos separados; odômetro com leituras próprias, sem regressão e com correção rastreável.
- **Alternativas:** campos únicos de peças e odômetro na moto ou no registro.
- **Consequência:** histórico auditável; troca de painel ainda não modelada.
- **Reversibilidade:** difícil após dados reais.
- **Dono:** equipe técnica.

### A5 — Viagem com segmentos e interrupção

- **Decisão:** viagem em segmentos, com estado `interrupted` recuperado na abertura.
- **Alternativas:** viagem como lista única de pontos.
- **Consequência:** pausas e quedas não geram distância.
- **Reversibilidade:** difícil após dados reais.
- **Dono:** equipe técnica.

### A6 — Vínculo entre viagem e sessão

- **Decisão:** vínculo explícito e opcional dos dois lados.
- **Alternativas:** vínculo obrigatório; associação só por horário.
- **Consequência:** aceita viagem sem CSV e CSV sem viagem.
- **Reversibilidade:** moderada.
- **Dono:** equipe técnica.

### A7 — Amostras largas com JSON

- **Decisão:** uma linha por instante, colunas tipadas para os campos normalizados e JSON para os valores como lidos.
- **Alternativas:** uma linha por campo.
- **Consequência:** consultas simples e mais espaço. O GPS do CSV mora só na amostra; o do aplicativo, só em `location_point`.
- **Reversibilidade:** moderada, porque o bruto pode ser relido.
- **Dono:** equipe técnica.

### A8 — Bruto imutável e derivado versionado

- **Decisão:** preservar bytes; `source_hash` é único no banco inteiro, confirmado pelo usuário em 2026-11-24; cada leitura registra a versão do parser; derivados recalculáveis.
- **Alternativas:** hash por moto; sobrescrever ao reimportar; guardar só o normalizado.
- **Consequência:** mais armazenamento, com auditoria e recálculo possíveis.
- **Reversibilidade:** **IRREVERSÍVEL** se os bytes não forem preservados desde a primeira importação.
- **Dono:** equipe técnica.

### A9 — Distância oficial

- **Decisão:** GPS válido, sem interpolar; roda e Rideology são comparativas (ADR 0001, complementado por D1).
- **Alternativas:** roda ou Rideology como oficial.
- **Consequência:** três valores lado a lado, com uma única fonte oficial.
- **Reversibilidade:** moderada; exige recalcular e revisar relatórios.
- **Dono:** usuário.

### A10 — Backup posterior e spikes antecipados

- **Decisão:** backup criptografado fica para fase posterior (ADR 0001); GPS com tela apagada e estrutura do CSV são verificados antes da F1.
- **Alternativas:** antecipar o backup; validar o GPS só na F2.
- **Consequência:** menos risco técnico na F1; perder o aparelho antes do backup perde os dados.
- **Reversibilidade:** a ordem é fácil de mudar; a perda de dados é **IRREVERSÍVEL**.
- **Dono:** usuário para o backup; equipe técnica para os spikes.

## Propostas técnicas

### P1 — Base React Native

- **Proposta:** Expo com build de desenvolvimento.
- **Alternativas:** React Native puro.
- **Justificativa:** configuração e permissões mais simples. O React Native puro é preferível se o spike S1 mostrar que o serviço de localização exige código nativo que o Expo não comporte.
- **Consequência:** dependência do ciclo de versões do Expo; suporte ao Android 17 [A VERIFICAR].
- **Reversibilidade:** moderada.
- **Dono:** usuário, com recomendação da equipe técnica.

### P2 — Biblioteca SQLite

- **Proposta:** começar a avaliação por `expo-sqlite` e trocar se ela falhar no spike S3.
- **Alternativas:** `op-sqlite` e `react-native-sqlite-storage`.
- **Justificativa:** integração esperada com P1 e porta isolada, que torna a troca possível.
- **Consequência:** a troca se limita ao adaptador; o formato do arquivo continua SQLite.
- **Reversibilidade:** moderada.
- **Dono:** equipe técnica.

Critérios de comparação, a medir no spike S3. Nada foi instalado e todo suporte específico é [A VERIFICAR].

| Critério | Peso | expo-sqlite | op-sqlite | react-native-sqlite-storage |
|---|---|---|---|---|
| SQLite 3.37+, STRICT e JSON | eliminatório | [A VERIFICAR] | [A VERIFICAR] | [A VERIFICAR] |
| Gatilhos e pragmas por conexão | eliminatório | [A VERIFICAR] | [A VERIFICAR] | [A VERIFICAR] |
| WAL e transação exclusiva | eliminatório | [A VERIFICAR] | [A VERIFICAR] | [A VERIFICAR] |
| Integração com P1 | alto | módulo do ecossistema Expo | módulo nativo de terceiros | módulo nativo de terceiros |
| Latência por lote | alto | medir | medir | medir |
| Criptografia opcional | médio | [A VERIFICAR] | [A VERIFICAR] | [A VERIFICAR] |
| Manutenção ativa | alto | [A VERIFICAR] | [A VERIFICAR] | [A VERIFICAR] |

### P3 — Validade do GPS separada do ponto

- **Proposta:** `gps_evaluation` versionada, fora das tabelas brutas.
- **Alternativas:** coluna de validade no ponto.
- **Justificativa:** limiares podem mudar sem alterar bruto, e a mesma regra serve às duas fontes.
- **Consequência:** uma linha derivada a mais por ponto avaliado.
- **Reversibilidade:** moderada.
- **Dono:** equipe técnica.

## Decisões abertas do usuário

| Tema | Dono | Reversibilidade | Prazo | Recomendação |
|---|---|---|---|---|
| Android mínimo | usuário | moderada | F1 | definir após o spike S3 |
| Distribuição | usuário | **IRREVERSÍVEL** após publicar | F1 | instalação direta no início |
| applicationId | usuário | **IRREVERSÍVEL** após publicar ou instalar com dados | F1 | fixar antes da primeira instalação |
| Retenção por prazo | usuário | **IRREVERSÍVEL** para o que for apagado | F1 | não apagar por prazo no MVP |
| Criptografia local | usuário | difícil após dados reais | F1 | avaliar ameaças antes do primeiro dado real |
| Fase da exportação manual | usuário | fácil | F1 | F1 |
| Licença | usuário | **IRREVERSÍVEL** para cópias já licenciadas | antes de distribuir | nenhuma até decidir |
| Visibilidade do repositório | usuário | **IRREVERSÍVEL** para cópias públicas | F0 | privado até revisar o histórico |
| Quem aprova PR | usuário | fácil | F0 | nomear uma pessoa |
| Quem aprova avanço de fase | usuário | fácil | F0 | nomear uma pessoa |
| Provedor e privacidade da IA | usuário | **IRREVERSÍVEL** para dados enviados | F5 | consentimento e dados mínimos |
| Limites do BLE | usuário | **IRREVERSÍVEL** para comando enviado | F6 | lista permitida só de leitura |

## Divergências consolidadas

| Tema | Plano técnico | Análise independente | Arbitragem |
|---|---|---|---|
| Viagem e telemetria | métrica ligada a um ou outro, sem vínculo entre eles | apontou a falta de vínculo | vínculo opcional dos dois lados (A6) |
| Pontos da viagem | ligados direto à viagem | segmentos e `interrupted` | segmentos e interrupção (A5) |
| Manutenção e odômetro | campos na moto e no registro | registros, itens, planos e leituras | modelo separado (A4) |
| Amostras | largas com JSON | largas ou longas, em aberto | largas com JSON (A7) |
| Dinheiro | custo e moeda, sem tipo | centavos inteiros com moeda | centavos com moeda (A3) |
| Tempo e ausência | UTC e ausência explícita | UTC com offset e motivo separado | offset e motivo (A2) |
| Importação | hash e URI em uma tabela, sem sessão | arquivo, sessão, amostra e problema | bruto separado da leitura versionada (A8) |
| Distância oficial | não definida | apontou a lacuna | ADR 0001 e D1 (A9) |
| Backup e exportação | fora do MVP inicial | antecipar para a F1 | backup posterior; exportação com fase aberta (A10) |
| Spikes | risco citado, sem spike | CSV, GPS e SQLite antes do modelo | spikes antes da F1 (A10) |
| Diretórios | por camada e funcionalidade, com `features` e `platform` | domínio, aplicação e adaptadores | quatro camadas com módulos (A1) |
| Achados | tabela de anomalias | tipos de medição a recomendação | `finding` com quatro tipos |
| Aceite da F0 | existência e coerência dos entregáveis | testes T0.1 a T0.6 | ambos, em `acceptance-criteria.md` |

## Reversibilidade do ADR 0001

O ADR 0001 não traz dono e reversibilidade por item. Esta nota os registra sem alterá-lo; o dono de todos é o usuário.

| Decisão do ADR 0001 | Reversibilidade |
|---|---|
| 1. CSV e trajetos reais fora do Git | regra fácil; publicação é **IRREVERSÍVEL** |
| 2. Fixtures sintéticas ou anonimizadas | fácil |
| 3. Dados locais no MVP | moderada |
| 4. Exportação manual no início | fácil |
| 5. Backup e sincronização posteriores | fácil |
| 6. Dispositivo de referência | fácil |
| 7. Distância oficial por GPS válido | moderada |

## Consequências gerais

O caminho sem rede é o principal, ao custo de mais migrações e recuperação local. Preservar bytes ocupa espaço e permite auditoria. As regras no banco deixam o esquema maior, mas impedem que um erro do aplicativo grave dado incoerente. P1 e P2 só se confirmam depois dos spikes.
