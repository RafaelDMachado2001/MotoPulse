# MotoPulse

Aplicativo para acompanhar viagens, manutenção e comportamento operacional de uma Kawasaki Ninja ZX-4R 2026.

O projeto combina registros de viagens, dados exportados pelo Rideology, histórico de manutenção e análises comparativas para ajudar o proprietário a entender melhor a motocicleta.

## Objetivos

- registrar viagens e rotas;
- acompanhar revisões, peças e custos;
- importar telemetria exportada pelo Rideology;
- comparar o comportamento da motocicleta ao longo do tempo;
- gerar relatórios claros sobre consumo, temperatura, RPM e desempenho;
- manter o funcionamento principal disponível sem conexão permanente.

## Estado do projeto

O projeto está na etapa de definição da fundação técnica. O desenvolvimento será dividido em fases independentes, com branches, commits e pull requests próprios.

## Princípio de execução

Cada fase possui:

1. objetivo;
2. entradas;
3. tarefas para os agentes;
4. artefatos esperados;
5. critérios de aceite;
6. decisão do responsável pelo projeto somente quando houver ambiguidade, risco ou mudança de escopo.

## Convenções de contribuição

Branches seguem Git Flow:

- `main`: versões estáveis;
- `develop`: integração contínua;
- `feature/<assunto>`: funcionalidade;
- `fix/<assunto>`: correção;
- `release/<versao>`: preparação de versão;
- `hotfix/<assunto>`: correção urgente em produção.

Commits usam linguagem natural e o formato Conventional Commits:

```text
tipo(escopo): descrição curta
```

Exemplos:

```text
feat(trips): add local trip persistence
fix(import): handle Rideology decimal values
docs(architecture): document offline data flow
test(telemetry): validate normalized measurements
```

Commits, branches, títulos de PR e mensagens públicas do projeto não devem mencionar ferramentas, modelos, agentes ou o processo interno de desenvolvimento.

Leia [PROJECT_PLAN.md](PROJECT_PLAN.md) e [ORCA_PROTOCOL.md](ORCA_PROTOCOL.md) antes de iniciar workers.
