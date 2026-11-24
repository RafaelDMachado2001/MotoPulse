# Plano mestre — MotoPulse

## Objetivo

Construir um aplicativo Android offline-first para registrar a Kawasaki Ninja ZX-4R 2026, acompanhar manutenção, registrar viagens, importar telemetria do Rideology, criar uma linha de base do comportamento da moto e gerar análises explicáveis com IA.

## Decisões atuais

- Plataforma inicial: Android.
- UI: React Native + TypeScript.
- Persistência local: SQLite.
- Backend: opcional e posterior.
- GPS: coleta adaptativa.
- Telemetria inicial: CSV exportado pelo Rideology.
- Bluetooth Rideology: investigação separada, somente leitura no início.
- IA: interpreta métricas e anomalias calculadas; não substitui medições determinísticas.
- Segurança: nenhuma escrita na ECU, painel, ABS, KTRC ou modos de pilotagem no MVP.
- Dispositivo de referência: Samsung Galaxy S25 Ultra, Android 17, One UI 9.0.
- Dados reais de viagens: ficam fora do Git; o repositório usa fixtures sintéticas ou anonimizadas.
- Persistência inicial: local, com exportação manual; backup criptografado será tratado em fase posterior.

## Fases

### F0 — Descoberta e fundação

**Objetivo:** transformar a ideia em contrato técnico executável.

**Entregáveis:** requisitos, arquitetura, modelo de dados, decisões registradas, estrutura inicial do repositório e critérios de aceite.

**Aceite:** Codex e Claude produzem relatórios independentes; divergências são consolidadas; decisões irreversíveis ficam explicitamente marcadas.

### F1 — Núcleo offline

**Objetivo:** criar a base do app sem integração com a moto.

**Escopo:** garagem, manutenção, SQLite, migrations, navegação, testes e dados de demonstração.

**Aceite:** criar/editar a moto, registrar manutenção e reabrir o app mantendo os dados em modo avião.

### F2 — Viagens e GPS adaptativo

**Objetivo:** registrar viagens reais com baixo consumo.

**Escopo:** serviço de localização em primeiro plano, amostragem adaptativa, pausa/retomada, classificação de paradas, rota e resumo.

**Aceite:** uma viagem pode ser interrompida, retomada e encerrada sem perder dados; pontos ficam persistidos localmente.

### F3 — Importação Rideology

**Objetivo:** fazer o primeiro dado real entrar no sistema.

**Escopo:** importador CSV, encoding, decimal, validação, normalização, métricas e gráficos.

**Aceite:** importar o CSV de 22/11/2026, preservar a origem e gerar métricas de distância, RPM, velocidade, marcha, temperatura e consumo com dados ausentes explicitamente marcados.

### F4 — Linha de base e anomalias

**Objetivo:** comparar a moto com o próprio histórico.

**Escopo:** viagens semelhantes, médias móveis, percentis, temperatura normal, RPM, consumo, intensidade e alertas.

**Aceite:** o sistema diferencia medição, desvio, hipótese e recomendação; não declara falha mecânica sem evidência suficiente.

### F5 — Relatórios com IA

**Objetivo:** transformar métricas em explicações úteis.

**Escopo:** relatório de viagem, semanal e mensal; hipóteses; recomendações; confiança; perguntas sobre o histórico.

**Aceite:** a IA recebe dados estruturados, cita evidências usadas e sinaliza incerteza.

### F6 — Bluetooth Rideology

**Objetivo:** investigar telemetria quase em tempo real.

**Escopo:** descoberta BLE, pareamento, leitura passiva, catálogo de serviços, comparação com CSV e integração somente leitura.

**Aceite:** não enviar comandos de controle; documentar incompatibilidades e fallback para CSV.

### F7 — Validação e evolução

**Objetivo:** preparar uso contínuo.

**Escopo:** testes em aparelho real, bateria, privacidade, exportação, backup, sincronização opcional e observabilidade.

**Aceite:** o app não perde registros em interrupções comuns e mantém o modo local como caminho principal.

## Regra de avanço

Uma fase só avança quando seus critérios de aceite estão verificados. Se Codex e Claude discordarem, o Orca cria uma tarefa de arbitragem; decisões de segurança, hardware, privacidade ou mudança de escopo permanecem com o usuário.
