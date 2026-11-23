# ADR 0001 — Dados de teste, privacidade e dispositivo de referência

## Status

Aceito

## Contexto

Os logs exportados pelo Rideology contêm nome de usuário, rota completa, horários e coordenadas GPS. Esses dados podem identificar deslocamentos reais e não devem fazer parte do histórico público do projeto nem ser enviados a serviços externos sem uma decisão específica.

O primeiro dispositivo de validação precisa ser conhecido para orientar testes de localização em segundo plano, permissões, consumo de bateria e comportamento do Android.

## Decisões

1. Arquivos CSV reais do Rideology e trajetos reais ficam fora do repositório Git.
2. O repositório utilizará fixtures sintéticas ou anonimizadas para testes automatizados e demonstrações.
3. O MVP armazenará os dados localmente no aparelho.
4. A exportação será manual no início.
5. Backup criptografado e sincronização serão tratados em uma fase posterior.
6. O dispositivo de referência será um Samsung Galaxy S25 Ultra com Android 17 e One UI 9.0.
7. A distância oficial será calculada a partir de pontos válidos de GPS; distância da roda e distância informada pelo Rideology serão preservadas como métricas comparativas.

## Consequências

- O importador precisa preservar o arquivo original localmente, mas não precisa copiá-lo para o projeto.
- Os testes de telemetria precisam cobrir valores ausentes, sentinelas e lacunas sem depender de uma viagem pessoal.
- O primeiro spike de GPS deve medir permissões, execução em segundo plano e consumo no Galaxy S25 Ultra.
- A aplicação não terá recuperação automática de dados entre aparelhos no MVP.
- A decisão de armazenamento remoto continua aberta para uma fase posterior.

## Critérios de verificação

- O Git não contém o CSV real nem coordenadas precisas da viagem.
- Existe uma fixture sintética com a mesma estrutura de colunas do Rideology.
- O cálculo de distância identifica a fonte e não interpola lacunas sem dados.
- O aplicativo funciona localmente no dispositivo de referência.
