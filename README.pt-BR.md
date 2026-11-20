<p align="center">
  <img src="docs/logo.png" alt="Logo do DatePlan" width="100">
</p>

<h1 align="center">DatePlan API</h1>

<p align="center">
  <b>Backend REST do DatePlan, o app para casais.</b><br>
  Spring Boot 4 · Java 21 · PostgreSQL · Redis · Firebase
</p>

<p align="center">
  <a href="https://github.com/DatePlanApplication/DatePlan">App</a> ·
  <a href="https://play.google.com/store/apps/details?id=mayckgomes.com.dateplan">Google Play</a> ·
  <a href="https://dateplanapp.netlify.app">Site</a> ·
  <a href="./README.md">English</a>
</p>

---

Este é o backend do [DatePlan](https://github.com/DatePlanApplication), onde o casal planeja encontros, guarda memórias e
conta os dias juntos. Duas contas se vinculam por **convite** e passam a compartilhar um calendário de **compromissos**,
**memórias** e o dia de início da relação. As mudanças chegam ao celular do parceiro por Firebase Cloud Messaging.

Roda em produção em uma VM pequena da Oracle Cloud (1 GB de RAM), por isso os padrões priorizam pouco uso de memória.

## Funcionalidades

- **Contas:** cadastro e login com e-mail e senha ou com **Google** (ID token validado pelo Firebase Admin SDK).
- **Sessões:** access token curto, refresh token longo com **rotação** e logout com **blacklist no Redis**.
- **Relacionamento:** vínculo entre dois usuários por convite (enviar, aceitar, recusar), data de início e encerramento.
- **Compromissos (dates):** criar, editar, listar e apagar; o parceiro é notificado a cada mudança.
- **Memórias:** criar, editar e apagar, ou **converter um compromisso passado em memória**.
- **Notificações push:** mensagens de dados via Firebase Cloud Messaging.
- **Consentimento:** guarda a versão da política de privacidade e dos termos aceita por cada usuário, e quando.

## Visão geral da API

Corpos de requisição e resposta estão em [`docs/`](./docs). Rotas com 🔒 exigem `Authorization: Bearer <accessToken>`.

| Módulo | Rotas |
|---|---|
| **Server** | `GET /` → `204` (health check usado pelo app) |
| **Auth** | `POST /auth/register` · `/auth/login` · `/auth/google` · `/auth/autoLogin` · `/auth/refreshToken` · 🔒 `/auth/logout` |
| **Users** 🔒 | `PATCH /users/changeName` · `/users/changePassword` · `/users/updateNotificationToken` · `/users/updatePolicyPrivacy` · `/users/updateTermsOfUse` · `DELETE /users/` |
| **Relationship** 🔒 | `GET /relationship/` · `PATCH /relationship/initialDay` · `DELETE /relationship/` |
| **Invites** 🔒 | `POST /invites/create` · `GET /invites/toMe` · `POST /invites/accept` · `POST /invites/decline` |
| **Dates** 🔒 | `POST /dates/create` · `PUT /dates/edit` · `DELETE /dates/delete` · `POST /dates/{id}/convert-to-memory` |
| **Memories** 🔒 | `POST /memories/create` · `PUT /memories/edit` · `DELETE /memories/delete` |

Os erros seguem um único formato: `{ "status": 404, "message": "..." }`.

## Autenticação

- **JWT** assinado com HMAC-SHA256, com `iss`, `aud`, `jti` único e a claim `type` (`Access` ou `Refresh`), conferida a cada verificação.
- Access token dura **15 minutos**; refresh token, **7 dias**. Cada refresh token vale **uma vez**: o `jti` vai para a blacklist
  do Redis e um novo par é emitido. No logout, os dois tokens vão para a blacklist até expirar.
- Senhas com **Argon2id** (`Argon2PasswordEncoder` do Spring Security); parâmetros vêm do ambiente.
- API sem estado: sem sessão e sem cookies. O `SecurityFilter` rejeita tokens expirados, inválidos ou na blacklist com `401`.

## Arquitetura

Camadas: `controllers` → `services` → `repositorys` → PostgreSQL, com Redis para a blacklist e Firebase Admin SDK para
login Google e push. Exceções de domínio (`exception/custom`) viram respostas HTTP em handlers `@ControllerAdvice`.
O schema é versionado com **Flyway** (`V1` a `V5`) e o Hibernate só valida (`ddl-auto=validate`). O pool do Hikari é
ajustado para instâncias pequenas (`maximum-pool-size=3`).

## Como rodar

**Pré-requisitos:** JDK 21, PostgreSQL, Redis e um JSON de **service account** do Firebase.

1. Suba PostgreSQL e Redis (por exemplo, com o `docker-compose.yml` mostrado no [README em inglês](./README.md#1-start-postgresql-and-redis)).
2. `cp .env.example .env` e preencha:

| Variável | Descrição |
|---|---|
| `SECRET` | Segredo HMAC dos JWTs (aleatório, 32 bytes ou mais) |
| `ISSUER` / `AUDIENCE` | Claims `iss` e `aud` |
| `SALT_LENGTH` · `HASH_LENGTH` · `PARALLELISM` · `MEMORY` · `ITERATIONS` | Parâmetros do Argon2id (padrão do Spring: `16`, `32`, `1`, `16384` KB, `2`) |
| `REDIS_URL` | Ex.: `redis://localhost:6379` |
| `FIREBASE_JSON_PATH` | Ex.: `file:/caminho/seguro/firebase.json` |
| `DB_URL` / `DB_USERNAME` / `DB_PASSWORD` | Conexão JDBC do PostgreSQL |

3. Rode:

```bash
./gradlew bootRun
```

A API sobe em `http://localhost:8080` (`curl -i http://localhost:8080/` deve responder `204`). O emulador Android acessa o
seu computador em `http://10.0.2.2:8080`.

> ⚠️ **Nunca faça commit** do `.env` nem do JSON do Firebase: quem tem esse arquivo age como o seu projeto Firebase.

**Docker:**

```bash
docker build -t dateplan-api .
docker run --env-file .env -p 8080:8080 dateplan-api
```

## Deploy

O [`cd.yml`](./.github/workflows/cd.yml) roda a cada push em `master`: gera o JAR, envia por SCP e reinicia o serviço
`systemd` por SSH. Precisa dos secrets `HOST`, `USERNAME` e `SSHKEY`. Em produção há um **NGINX** na frente (TLS e rate limit),
configurado no servidor e fora deste repositório.

## Testes

Existe apenas um teste de contexto (`contextLoads`), que exige PostgreSQL, Redis e as variáveis acima.

## Roadmap e limitações

- [ ] Testes automatizados dos services e do fluxo de autenticação
- [ ] `docker-compose.yml` para desenvolvimento no repositório
- [ ] Documentação OpenAPI / Swagger
- [ ] Rate limit na própria aplicação (hoje é feito pelo NGINX)
- [ ] Remover a rota do console H2 da configuração de segurança e a dependência do H2 console
- [ ] Usar logger no lugar de `System.out`

## Contribuindo e licença

Veja o [CONTRIBUTING.md](./CONTRIBUTING.md), o [Código de Conduta](./CODE_OF_CONDUCT.md) e o [SECURITY.md](./SECURITY.md).
Código *source-available* sob a [PolyForm Noncommercial 1.0.0](./LICENSE): uso **não comercial** permitido; uso comercial
exige autorização por escrito. O nome "DatePlan", o logotipo e os ícones não estão cobertos pela licença.

Criado por [Mayck Gomes](https://github.com/MayckGomes).
