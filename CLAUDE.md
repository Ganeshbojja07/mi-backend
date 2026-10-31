# mi-backend

Spring Boot service for the motor insurance automation: mail ingestion, Case IDs, the Temporal workflow, Java activities, REST API for the UI, Flyway migrations, and the cross-repo contracts. See `../CLAUDE.md` for how this fits with `mi-ai-worker` and `mi-ui`.

## Stack

Java 21 · Spring Boot (version supported by `temporal-spring-boot-starter`) · Temporal Java SDK · Spring Data JPA · Flyway · MySQL 8 · AWS SDK v2 S3 (MinIO locally) · Gmail API client · Jakarta Mail (MIME) · Thymeleaf (email templates) · Spring Security · Apache Commons Text (Jaro-Winkler) · Logback + logstash-logback-encoder · JUnit 5 · Testcontainers (MySQL) · `temporal-testing`.

## Commands

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=local   # run (needs infra + Temporal dev server)
./mvnw test                                                # unit + workflow tests
./mvnw verify                                              # + integration tests (Testcontainers, needs Docker)
./mvnw spotless:apply                                      # format
docker compose -f infra/docker-compose.yml up -d           # MySQL + MinIO + bucket init
```

## Layout

```
contracts/                  # SOURCE OF TRUTH — activity JSON Schemas + openapi.yaml
infra/                      # docker-compose.yml, .env.example
local-mail/{inbox,outbox}/  # .eml files when MAIL_PROVIDER=local (gitignored)
src/main/java/ae/mi/backend/
  config/       # Temporal, S3, security, Jackson, MDC
  ingestion/    # MailSource (Gmail | local), MailRouter (new case vs reply), CaseIdGenerator
  workflow/     # ApplicationProcessingWorkflow (+Impl), signals, query, WorkflowConfig
  activities/   # JavaActivities (+Impl), AiActivities (interface only — implemented in Python)
  rules/        # ValidationRule implementations, RuleEngine, rules config
  mail/         # MailSender (Gmail | local), Thymeleaf templates
  storage/      # S3 storage service, pre-signed URLs
  persistence/  # JPA entities + repositories
  api/          # REST controllers, DTOs, SSE
  audit/        # StageLog activity interceptor
src/main/resources/
  db/migration/ # Flyway V1__..., never edit an applied migration
  templates/mail/
  application.yml, application-local.yml
```

## Conventions

- **Workflow determinism:** workflow classes use only `Workflow.*` APIs (`Workflow.sleep`, `Workflow.await`, `Workflow.newTimer`, `Workflow.currentTimeMillis`). No Spring beans, repositories, `LocalDate.now()`, `UUID.randomUUID()` or normal loggers (use `Workflow.getLogger`).
- **Cross-language activities:** `AiActivities` is an `@ActivityInterface` whose methods set `@ActivityMethod(name = "...")` to the exact camelCase names used in Python; its stub uses task queue `ai-activities`.
- **Idempotency:** activities upsert by natural keys (`case_id`, `round`, `stage`, `rule_code`); `email_message.gmail_message_id` is unique.
- **Case IDs:** `APP-<year>-<6-digit seq>` from a `case_counter` table (one row per year) using `SELECT ... FOR UPDATE` in a transaction.
- **Logging:** put `caseId`, `round`, `workflowId` in MDC for API requests and activity executions; JSON logs. Mask Emirates ID numbers everywhere.
- **Validation rules** implement `ValidationRule { code(); severity(); evaluate(ApplicationData) }`; thresholds live in `application.yml` under `mi.rules.*`, never hard-coded. Every failure carries a user-facing `fixHint`.
- **API:** controllers match `contracts/openapi.yaml` exactly; update the spec first, then code. Errors use RFC 7807 problem details.
- **Contracts:** after changing `contracts/`, mi-ai-worker and mi-ui must regenerate their models (say so in `../PROGRESS.md`).

## Testing expectations

- Every `ValidationRule`: unit tests for pass, fail and edge cases (missing data, low confidence).
- Workflow: `TestWorkflowEnvironment` with time skipping and mocked activities — happy path, correction loop, reply after timeout, max rounds, review signal, duplicate signals.
- Repositories / Flyway: Testcontainers MySQL integration tests.
- Mail: `MailRouter` tests with `.eml` fixtures (new application, reply by threadId, reply by Case ID in subject, unrelated mail).

## Don'ts

- Don't call the Python worker over HTTP or edit its code from here except via contracts.
- Don't store file bytes in MySQL or in Temporal payloads — S3 keys only.
- Don't let any LLM output change a decision.
- Don't commit `.env`, OAuth tokens, `local-mail/`, or real documents.
