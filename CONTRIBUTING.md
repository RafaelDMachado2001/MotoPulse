# Contribuição

## Fluxo de trabalho

1. Atualize a branch local a partir de `develop`.
2. Crie uma branch curta com prefixo Git Flow.
3. Faça commits pequenos, completos e relacionados a uma única mudança.
4. Execute os testes e registre o resultado no pull request.
5. Abra o pull request contra `develop`.
6. Faça a integração somente após revisão e critérios de aceite atendidos.

## Commits

Use Conventional Commits, em inglês, com descrição objetiva:

```text
feat(scope): add capability
fix(scope): correct behavior
docs(scope): update documentation
test(scope): cover scenario
refactor(scope): simplify implementation
chore(scope): update project setup
```

Não mencione ferramentas, modelos, agentes, automação ou autoria assistida em commits, branches ou pull requests.

## Pull requests

Cada fase deve resultar em um ou mais pull requests pequenos. Um PR deve conter:

- objetivo;
- escopo incluído;
- escopo não incluído;
- decisões relevantes;
- testes executados;
- limitações conhecidas;
- evidência visual quando houver mudança de interface.

## Qualidade

- preserve os dados brutos de telemetria;
- não misture refatoração ampla com uma funcionalidade específica;
- não inclua credenciais ou arquivos pessoais;
- documente mudanças de esquema do banco;
- mantenha o aplicativo utilizável offline sempre que o requisito permitir.
