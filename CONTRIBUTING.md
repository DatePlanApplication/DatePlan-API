# Contribuindo com a DatePlan API

Obrigado pelo interesse! Contribuições são bem-vindas.

## Antes de começar
- Procure nas [issues](../../issues) se o assunto já existe. Para mudanças grandes, abra uma issue antes para alinharmos a ideia.
- Configure o ambiente seguindo o [README](./README.md).

## Fluxo
1. Faça um fork e crie uma branch (`feature/minha-ideia` ou `fix/meu-bug`).
2. Faça commits pequenos e com mensagens claras, de preferência em [Conventional Commits](https://www.conventionalcommits.org/).
3. Garanta que `./gradlew build -x test` passa e, se você adicionou testes, `./gradlew test`.
4. Abra um Pull Request preenchendo o template.

## Boas práticas
- Siga a organização em camadas: `controllers` → `services` → `repositorys`. Regras de negócio ficam nos services.
- Erros de domínio viram exceções em `exception/custom` e são traduzidos por um handler em `exception/handler`.
- Mudanças no banco entram como **novas** migrations Flyway (`V6__...`). Nunca edite uma migration já existente.
- Endpoints novos ou alterados devem atualizar o arquivo correspondente em `docs/`.
- Não inclua segredos, `.env`, o JSON do Firebase, senhas nem tokens reais.
- Não registre dados pessoais ou tokens em logs.

## Licença das contribuições

Ao enviar um pull request, você declara que é autor do que está enviando e concorda que a contribuição seja
licenciada sob a [PolyForm Noncommercial 1.0.0](./LICENSE). Você também autoriza o mantenedor a relicenciar o
projeto no futuro, incluindo para uso comercial.
