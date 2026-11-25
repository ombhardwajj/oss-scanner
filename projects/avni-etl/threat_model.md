# Threat model

The scanner reads this file before it starts. It covers avni-etl.

## What this project does and where untrusted input enters

Avni is an open-source platform that NGOs and governments use for field work in health, nutrition, water, education
and social welfare. Field workers collect data on an offline-first Android app that syncs with a central server. The
server stores personal and health data about the people those organisations serve, and one deployment holds many
organisations (tenants). Reading or changing another organisation's data is the most serious class of finding in
every Avni component.

avni-etl (Java 17, Spring Boot, PostgreSQL) reads the transactional tables that avni-server writes and builds, for
each organisation, a reporting schema of flat tables and views in the same database. It runs as a scheduled job per
organisation (Quartz) and also serves a small HTTP API used by the web app (job management and media search). It
connects to the database as the owner of the source tables and switches to the organisation's database role where it
needs tenant isolation, so any SQL that runs without that role, or that can be made to drop it, reaches every
organisation.

Untrusted input reaches it through:

- **Metadata that organisation administrators author in avni-server**: names and uuids of concepts, forms, subject
  types, programs, encounter types, address-level types, and the organisation's schema name and database user. The
  ETL turns these into table names, column names, view definitions and SQL literals. All of them must be treated as
  attacker-controlled strings: avni-server does not restrict the characters in names, and uuids are stored as sent.
- **Observation data** written by field users (JSON values, media URLs, text).
- **Rows that an organisation's own database role can write**, including the ETL's own metadata tables and scheduler
  tables in the public schema.
- **The HTTP API**: a Cognito or Keycloak JWT in the `AUTH-TOKEN` header, request parameters and JSON bodies of the
  media search and job endpoints.

### Trust model

- Unauthenticated clients are untrusted. Field users and organisation administrators are trusted only inside their
  own organisation; an organisation can be created by self-signup, so "administrator of some organisation" is a
  position an attacker can hold. Account administrators are trusted only for their own organisations.
- Avni's operators, `application.properties` and the environment are trusted.

## Components that matter most / least

### Highest priority

1. Every place SQL is assembled: the StringTemplate files under `src/main/resources/sql/`, the `repository/sync`
   and `repository/sql` packages, the `domain/metadata/diff` classes (DDL), and any `jdbcTemplate` call that takes
   a name, a uuid or a request value.
2. Role handling: `JdbcContextWrapper`, schema and role creation in `SchemaMigrationService`, grants, and every
   statement that runs outside an organisation context.
3. Authentication and authorisation of the HTTP API (`AuthenticationFilter`, the IAM services, `UserRepository`,
   the controllers), including which organisation a request may act on.
4. Media search and S3 URL signing.
5. The Quartz scheduler tables and job data.

### Lower priority and out of scope

- Out of scope: tests (`src/test/**`), `deploy/`, `makefiles/`, CI configuration.

## How to exercise it

- `/src` is the checkout. The image ran `./gradlew clean build testClasses -x test` and then `./gradlew test`, so
  Gradle and all dependencies are cached. Use `--offline`.
- PostgreSQL 16 is installed but not running. `avni-test-db` starts it and makes sure the `openchs_test` database
  exists with avni-server's schema (user `openchs`, password `password`); avni-server's migrations are at
  `/opt/avni-server`. `avni-test-db reset` gives a fresh database.
- Run one test class with `./gradlew --offline test --tests "org.avniproject.etl.<package>.<Class>"`. Integration
  tests load SQL fixtures from `src/test/resources` and run the real sync against the database; add a fixture with a
  hostile name or uuid to show an injection, and print the SQL the ETL generated.

## How you rate severity

- **Critical**: SQL or data access that crosses organisations or runs as the database owner, by anyone, including an
  organisation administrator acting through metadata; unauthenticated access to any organisation's data; acting as
  another user.
- **High**: SQL injection that stays inside the caller's own organisation; reading secrets; an organisation's
  database role gaining the owner's rights.
- **Medium**: a missing privilege check limited to the caller's own organisation; tokens or personal data in logs;
  one organisation stalling the ETL for all.
- **Low**: information disclosure without personal data or secrets; hardening gaps with no demonstrated outcome.

## Anything to leave alone

- Development defaults that deployments replace (default passwords, keys and URLs in configuration templates and
  Makefiles). Do report a default that is used where no setting exists to replace it.
- Vulnerabilities in third-party dependencies unless this project's code reaches them in an exploitable way.
- Missing rate limiting and missing security headers, unless they combine into one of the outcomes above.

### Reports and patches

- One report per root cause, listing every affected endpoint, file or class.
- State who the attacker is (unauthenticated, field user, organisation administrator) and whether the effect crosses
  organisations.
- Include a reproducer (a test beside the existing ones, or the exact requests) and a minimal patch with a
  regression test.
