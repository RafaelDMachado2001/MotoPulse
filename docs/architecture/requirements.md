# Requisitos

| ID | Requisito | Fase | Teste |
|---|---|---|---|
| RF-01 | Cadastrar e editar motocicleta | F1 | T1.1 |
| RF-02 | Registrar manutenção, itens, planos e odômetro | F1 | T1.6 |
| RF-03 | Criar, pausar, retomar, recuperar e finalizar viagem | F2 | T2.1,T2.6 |
| RF-04 | Persistir GPS e calcular distância oficial sem interpolar lacunas | F2 | T2.5 |
| RF-05 | Importar CSV preservando bytes, hash, origem e issues | F3 | T3.1–T3.5 |
| RF-06 | Associar opcionalmente viagem e sessão de telemetria | F3 | T3.7 |
| RF-07 | Calcular métricas versionadas e comparativas | F3/F4 | T3.7,T4.4 |
| RF-08 | Produzir achados separados em medição, desvio, hipótese e recomendação | F4/F5 | T4.3 |
| RF-09 | Gerar relatório com evidências e incerteza | F5 | T5.1,T5.2 |
| RF-10 | Funcionar sem conexão e sem BLE | F1–F4 | T1.2,T5.4 |
| RNF-01 | Dados locais são fonte de verdade e transações não expõem estado parcial | F1 | T1.3,T3.5 |
| RNF-02 | UUID, UTC+offset, unidades canônicas, NULL com motivo e dinheiro em centavos | F0/F1 | T0.3 |
| RNF-03 | Bruto imutável e derivados recalculáveis/versionados | F3 | T3.1,T4.4 |
| RNF-04 | Não armazenar CSV/rotas reais no Git | F0 | T0.6 |
| RNF-05 | Não escrever em ECU, ABS, KTRC ou painel | F6 | T6.1,T6.2 |
| RNF-06 | Interrupção/reabertura recupera viagem sem apagar dados | F2 | T2.1,T2.2 |

Offline: cadastro, manutenção, odômetro, persistência/consulta de viagens e GPS, importação local, métricas determinísticas e relatórios baseados em dados locais. Fora de escopo: backend/sincronização, backup criptografado, BLE, escrita na moto, exportação manual (proposta F1; decisão do usuário aberta), IA remota e diagnóstico mecânico.

Lacunas abertas: Android mínimo, distribuição, applicationId, retenção, criptografia local, licença/visibilidade, aprovador e avanço de fase, provedor/privacidade da IA (F5), limites BLE (F6), colunas/unidades reais do CSV, limiares de anomalia e metas de bateria.
