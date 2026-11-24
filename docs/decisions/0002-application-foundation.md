# ADR 0002 — Fundação da aplicação

## Status

Proposta para revisão; decisões do usuário permanecem abertas.

## Decisões técnicas propostas

| Tema | Proposta e alternativas | Consequência/reversibilidade | Dono |
|---|---|---|---|
| Camadas | domain/application/adapters/ui; alternativa monolítica | Contratos testáveis; reversível antes de F1 | Equipe técnica |
| UI | React Native + TypeScript; Expo ou RN puro [A VERIFICAR] | Expo acelera, RN puro dá controle nativo; escolha reversível com custo | Usuário + equipe |
| SQLite | SQLite local; biblioteca [A VERIFICAR] | Offline e transações; trocar biblioteca exige migração | Usuário + equipe |
| Dados | UUID, UTC+offset, SI, NULL com motivo, bruto imutável | Evita perda semântica; cara de mudar após dados reais | Usuário |
| Telemetria | amostras largas + JSON extra | Consultas simples e evolução; campos reais aguardam CSV | Equipe técnica |
| Distância | GPS válido oficial; roda/Rideology comparativas | Coerente com ADR 0001; reversível só com migração | Usuário |
| Viagem/sessão | vínculo opcional nos dois sentidos; estado interrupted | Recuperação explícita; reversível | Usuário |

## Decisões abertas do usuário

Android mínimo, distribuição, applicationId (irreversível após publicação/instalação), retenção e exclusão, criptografia local, exportação manual (recomendação F1), licença e visibilidade (licença pública é irreversível nas cópias), quem aprova PR e avanço de fase, provedor/privacidade/consentimento/custo da IA em F5, limites e termos do BLE em F6. Cada item requer decisão explícita; nenhum é fechado aqui.

Backup criptografado continua posterior conforme ADR 0001. Spikes de GPS com tela apagada no Galaxy S25 Ultra e estrutura do CSV com fixture sintética são pré-F1 e apenas documentados nesta fase.
