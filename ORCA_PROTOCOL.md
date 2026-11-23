# Protocolo de execução — ZX-4R Intelligence

## Organização

O Orca coordena o trabalho entre os dois ambientes, mas o repositório deve permanecer independente das ferramentas usadas para desenvolvê-lo. Nenhum artefato público, commit, branch ou pull request deve mencionar os agentes ou o processo de orquestração.

## Papéis internos

### Codex/GPT — arquiteto e integrador

- define contratos e interfaces;
- transforma decisões em tarefas implementáveis;
- implementa ou integra mudanças;
- executa testes e verifica critérios de aceite;
- consolida o resultado final da fase.

### Claude Code — implementador/revisor independente

- propõe uma solução independente para a fase;
- implementa quando designado pelo Orca;
- revisa riscos, casos de borda e qualidade;
- tenta quebrar os critérios de aceite;
- reporta divergências com evidências.

### Orca — coordenador

- inicia workers por fase;
- passa especificações e artefatos, não contexto implícito;
- aguarda resultados;
- dispara revisão cruzada;
- mantém o estado da fase;
- só solicita decisão humana em pontos marcados.

## Fluxo automático de cada fase

```text
1. Orca lê PROJECT_PLAN.md
2. O ambiente principal produz o plano técnico da fase
3. O ambiente de revisão produz análise independente
4. Orca entrega os dois resultados para revisão cruzada
5. O ambiente principal consolida decisões e implementação
6. O ambiente de revisão executa QA e tenta encontrar falhas
7. Orca verifica critérios de aceite
8. Se aprovado: registra fase concluída e inicia a próxima
9. Se bloqueado: cria decisão para o usuário
```

## Contrato de saída de cada worker

Cada worker deve retornar:

```text
STATUS: ready | needs_decision | blocked | done
PHASE: F<n>
SUMMARY: resumo curto
CHANGES: arquivos criados ou modificados
EVIDENCE: comandos, testes ou observações verificáveis
RISKS: riscos ainda abertos
DECISIONS: decisões tomadas ou necessárias
NEXT: próximo passo recomendado
```

## Regras de segurança

- não expor credenciais, tokens ou dados pessoais;
- não alterar ECU, ABS, KTRC ou comandos de pilotagem;
- não executar ações destrutivas sem aprovação explícita;
- não tratar hipótese da IA como diagnóstico confirmado;
- não instalar dependências ou enviar dados externos sem autorização adequada;
- preservar o CSV original e os dados brutos;
- toda migração de banco deve ser reversível ou acompanhada de backup.

## Regras de Git

- uma fase pode conter várias branches e pull requests;
- branches devem seguir as convenções do `CONTRIBUTING.md`;
- o branch de destino normal é `develop`;
- commits devem usar linguagem natural e Conventional Commits;
- commits, branches e PRs não podem mencionar IA, modelos, agentes, Codex, Claude ou Orca;
- o histórico público deve parecer o histórico normal de uma equipe de software;
- mudanças de documentação e código podem ser separadas quando isso melhorar a revisão.

## Template de especificação para o Orca

```text
Projeto: ZX-4R Intelligence
Fase: F<n> — <nome>
Objetivo: <resultado verificável>
Entradas: <arquivos, decisões e dados>
Restrições: <segurança, plataforma e escopo>
Tarefas do Codex: <lista>
Tarefas do Claude: <lista>
Artefatos esperados: <lista>
Critérios de aceite: <lista>
Não fazer: <limites>
Formato de saída: seguir o contrato STATUS/PHASE/SUMMARY/CHANGES/EVIDENCE/RISKS/DECISIONS/NEXT.
```
