# Requisitos

| ID | Descrição verificável | Fase | Teste |
|---|---|---|---|
| RF-01 | Criar/editar uma motocicleta e reabri-la | F1 | T1.1 |
| RF-02 | Registrar registro de manutenção com custo/moeda opcionais | F1 | T1.2 |
| RF-03 | Registrar itens de manutenção ligados a registro | F1 | T1.3 |
| RF-04 | Criar plano com intervalo de km/tempo e vincular execução | F1 | T1.4 |
| RF-05 | Registrar odômetro e correção regressiva rastreável | F1 | T1.5 |
| RF-06 | Ler dados de demonstração sintéticos offline | F1 | T1.6 |
| RF-07 | Criar/pausar/retomar/finalizar viagem | F2 | T2.1 |
| RF-08 | Recuperar viagem interrupted após reabertura | F2 | T2.2 |
| RF-09 | Persistir GPS e distância válida sem interpolar lacunas | F2 | T2.3 |
| RF-10 | Importar bytes Rideology, hash, origem e issues | F3 | T3.1 |
| RF-11 | Associar opcionalmente viagem e sessão | F3 | T3.2 |
| RF-12 | Calcular métricas versionadas e distâncias comparativas | F3/F4 | T3.3 |
| RF-13 | Emitir findings classificados com evidência | F4 | T4.1 |
| RF-14 | Gerar relatório com versão, evidência e incerteza | F5 | T5.1 |
| RF-15 | Exportar manualmente dados conforme fase decidida | F1/F7 | T7.1 |
| RF-16 | Ler fixture sintética com as 14 colunas do ADR 0001, preservando constantes e N | F1/F3 | T3.5 |
| RF-17 | Exibir navegação entre garagem, manutenção, viagens e relatórios | F1 | T1.10 |
| RF-18 | Amostrar GPS adaptativamente e classificar paradas | F2 | T2.4 |
| RF-19 | Gerar gráficos de métricas importadas | F3 | T3.6 |
| RF-20 | Gerar relatórios semanal/mensal e perguntas sobre histórico | F5 | T5.2 |
| RNF-01 | SQLite local é fonte de verdade e transações não expõem estado parcial | F1 | T1.7 |
| RNF-02 | IDs UUID, UTC/offset, unidades canônicas e ausência explícita | F0/F1 | T0.3 |
| RNF-03 | Bruto imutável e derivados recalculáveis/versionados | F3 | T3.4 |
| RNF-04 | CSV/rotas reais não entram no Git; fixtures sintéticas são permitidas | F0 | T0.6 |
| RNF-05 | Nenhuma escrita em ECU, ABS, KTRC ou painel | F6 | T6.1 |
| RNF-06 | Reabertura não perde registros persistidos | F1/F2 | T1.8 |
| RNF-07 | Capacidades locais principais funcionam offline | F1–F4 | T1.9 |

Offline: demonstração, moto, manutenção, odômetro, viagens/GPS, importação local, métricas determinísticas, consulta e relatórios locais. Fora de escopo F0/F1: backend, sincronização, backup criptografado (posterior ADR 0001), BLE, escrita na moto e IA remota. Exportação manual faz parte do produto; a fase permanece aberta (recomendação F1).

Lacunas: Android mínimo, distribuição, applicationId, retenção, criptografia, licença/visibilidade, aprovação/avanço, provedor e privacidade IA (F5), limites BLE (F6), colunas/unidades CSV e limiares GPS/anomalia [A VERIFICAR].
