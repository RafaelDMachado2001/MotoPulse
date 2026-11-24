# ADR 0002 — Fundação da aplicação

## Status

Proposta para revisão; ADR 0001 prevalece.

## Contexto

MotoPulse precisa de uma base Android offline-first, com dados brutos Rideology preservados, GPS sem interpolação e derivados recalculáveis. O CSV real e algumas decisões de produto ainda não existem; portanto campos desconhecidos ficam [A VERIFICAR] e não se fecha decisão irreversível prematuramente.

## Decisões técnicas propostas

| Decisão | Alternativas | Recomendação e justificativa | Consequência/reversibilidade | Dono |
|---|---|---|---|---|
| Camadas domain/application/adapters/ui | monólito, MVC | camadas: isolamento facilita testes offline | reversível antes de F1; depois custo médio | equipe técnica |
| React Native + TypeScript | Expo ou RN puro | **PROPOSTA: Expo** para velocidade, permissões e build; RN puro se spike exigir módulo nativo não suportado | trocar depois exige configuração/build, reversível com custo | usuário/equipe |
| SQLite | biblioteca SQLite [A VERIFICAR], armazenamento remoto | **PROPOSTA: biblioteca SQLite madura com migrations explícitas**; escolher pela transação, WAL, FKs e suporte Android, sem instalar nesta fase | trocar exige adapter/migration; dados permanecem SQLite | usuário/equipe |
| UUID, UTC+offset, SI e NULL motivado | inteiros, horário local, zero-sentinela | proposta arbitrada: evita colisões e perda semântica | cara de mudar após dados reais | usuário |
| Amostra larga + JSON extra | modelo longo/EAV | larga para consultas previsíveis; JSON absorve campos desconhecidos | recalculável, migração possível após CSV | equipe técnica |
| Bruto imutável + SHA global | hash por moto, sobrescrever | hash global deduplica reimportação e bytes sempre preservados | irreversível na prática após primeira importação | usuário |
| Distância GPS oficial | roda/Rideology oficial | ADR 0001: GPS válido oficial; demais comparativas | reversível só por migração/relatórios | usuário |
| Trip/session opcionais | vínculo obrigatório | cobre viagem sem CSV e CSV sem GPS | reversível | usuário |

## Consequências

Offline é o caminho principal, com maior complexidade de migrations e recuperação. Preservar bytes aumenta armazenamento, mas permite auditoria e recálculo. A escolha Expo/SQLite permanece proposta até validação; nenhum spike ou dependência é executado na F0.

## Decisões abertas do usuário

| Tema | Dono | Reversibilidade | Prazo | Recomendação |
|---|---|---|---|---|
| Android mínimo | usuário | média | F1 | decidir antes de publicar |
| Distribuição | usuário | **IRREVERSÍVEL após publicação** | F1 | definir APK/loja |
| applicationId | usuário | **IRREVERSÍVEL após instalação/publicação** | F1 | fixar antes do primeiro build |
| Retenção/exclusão | usuário | média | F1 | política mínima local explícita |
| Criptografia local | usuário | média | F1/F7 | avaliar threat model |
| Fase da exportação manual | usuário | média | F1 (recomendado) | antecipar para reduzir perda |
| Licença | usuário | **IRREVERSÍVEL nas cópias licenciadas** | F0/F1 | decidir antes de distribuir |
| Visibilidade do repositório | usuário | **IRREVERSÍVEL para cópias públicas** | F0 | escolher privado/público |
| Aprovação de PR/avanço | usuário | baixa | F0 | nomear aprovador |
| Provedor/privacidade da IA | usuário | alta; envio de dados pode ser irreversível | F5 | consentimento e dados mínimos |
| Limites/privacidade BLE | usuário | **envio de comando potencialmente irreversível** | F6 | somente leitura, lista permitida |

Backup criptografado continua posterior conforme ADR 0001. Spikes pré-F1 são documentados, não executados aqui.

## Bibliotecas SQLite — proposta, sem instalação

`expo-sqlite`, `react-native-sqlite-storage` e `op-sqlite` são candidatas. Comparar: suporte a migrations explícitas, transações, WAL, foreign keys, STRICT, TypeScript, Android mínimo (a definir), manutenção e observabilidade; versões e suporte exato ficam [A VERIFICAR]. **Proposta:** começar com Expo SQLite se os testes de schema atenderem; caso contrário avaliar RN SQLite ou op-sqlite. Nenhuma escolha é final nesta fase.

## Escala de reversibilidade

Alta = troca sem migração de dados; média = migração/configuração; baixa = retrabalho significativo; irreversível = publicação, distribuição, envio de dados ou perda de bruto. “Média/alta” nas tabelas anteriores deve ser lida por esta escala.

## Divergências arbitradas

| Tema | Análise/plano divergentes | Arbitragem |
|---|---|---|
| CSV | modelo presumido versus estrutura desconhecida | usar fatos de csv-structure.md; desconhecidos [A VERIFICAR] |
| Amostras | larga versus longa | larga + JSON extra, com bruto imutável |
| Distância | GPS, roda e Rideology | ADR 0001: GPS válido oficial; roda/Rideology comparativas |
| Exportação | posterior versus perda de dados | produto inclui; fase aberta, recomendação F1 |
| Fixture | inexistente versus necessária | fixture sintética de 14 colunas atribuída a F1/F3, T3.5 |

## Aberturas adicionais

| ID | Questão | Dono | Reversibilidade | Prazo/recomendação |
|---|---|---|---|---|
| D1 | GPS CSV congelado/salto/sem sinal entra na distância oficial? Prioridade se houver GPS app? ADR 0001 fixa GPS válido, mas não cobre esta origem | usuário | média | F3; manter comparativa até decisão |
| D2 | Exclusão apaga bruto? | usuário | irreversível se apagar | F1/F7; recomendar RESTRICT |
| D3 | Nome de usuário/apelido no bruto | usuário | alta; vazamento é irreversível | F3; proposta: bruto local, fora de logs/exportação/IA |
| Manutenção/odômetro/centavos | preservar registros, correções e centavos+moeda | usuário/equipe | baixa após dados | F1; manter schema proposto |
