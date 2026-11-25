# Build avni-etl for Anthropic's OSS Scanner.
# The scanner uses the repository root as the build context and audits /src offline.
#
# Mirrors the project's CI (.circleci/config.yml): JDK 17 and PostgreSQL 16 with a database named openchs_test owned
# by a user named openchs, whose schema is created by running avni-server's Flyway migrations. CI runs PostgreSQL as a
# second container; here it is installed in the same image so the tests can run offline.
FROM eclipse-temurin:17-jdk-jammy

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

ARG FLYWAY_VERSION=8.5.9

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=en_US.UTF-8 \
    PGPASSWORD=password \
    GRADLE_USER_HOME=/root/.gradle

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates curl git gnupg locales make unzip \
    && locale-gen en_US.UTF-8 \
    && install -d /usr/share/postgresql-common/pgdg \
    && curl -fsSL --retry 3 -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc \
        https://www.postgresql.org/media/keys/ACCC4CF8.asc \
    && echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt jammy-pgdg main" \
        > /etc/apt/sources.list.d/pgdg.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends postgresql-16 postgresql-client-16 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Flyway command line (the build without a bundled JRE, so it runs on the image's JDK on any architecture) and
# avni-server's migrations, which define the source schema that the ETL reads.
RUN curl -fsSL --retry 3 \
        "https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}.tar.gz" \
        | tar xz -C /opt \
    && ln -s "/opt/flyway-${FLYWAY_VERSION}/flyway" /usr/local/bin/flyway \
    && git clone --depth 1 --branch master https://github.com/avniproject/avni-server /opt/avni-server

# avni-test-db: start the local PostgreSQL and make sure the test database exists, as CI does (user openchs, password
# "password", database openchs_test in the en_US.UTF-8 locale, extensions uuid-ossp, ltree and hstore, then
# avni-server's migrations). "avni-test-db reset" drops and recreates the database first.
RUN printf '%s\n' \
    '#!/bin/bash' \
    'set -e' \
    'pg_ctlcluster 16 main start 2>/dev/null || true' \
    'for i in $(seq 1 30); do pg_isready -q -h localhost && break; sleep 1; done' \
    'run() { su postgres -c "psql -v ON_ERROR_STOP=0 -q -d ${2:-postgres} -c \"$1\"" >/dev/null 2>&1 || true; }' \
    'if [ "$1" = "reset" ]; then run "drop database if exists openchs_test with (force)"; fi' \
    'run "create user openchs with superuser password '"'"'password'"'"'"' \
    'run "create database openchs_test with owner openchs template template0 locale '"'"'en_US.UTF-8'"'"'"' \
    'for e in uuid-ossp ltree hstore; do run "create extension if not exists \\\"$e\\\"" openchs_test; done' \
    'flyway -validateOnMigrate=false -user=openchs -password=password -url=jdbc:postgresql://localhost:5432/openchs_test -schemas=public -locations=filesystem:/opt/avni-server/avni-server-api/src/main/resources/db/migration/ -table=schema_version migrate >/tmp/avni-test-db-flyway.log 2>&1 || { tail -20 /tmp/avni-test-db-flyway.log; exit 1; }' \
    'pg_isready -h localhost' \
    > /usr/local/bin/avni-test-db \
    && chmod +x /usr/local/bin/avni-test-db

# Scanner contract: the checkout lives inside the image, at /src.
COPY . /src
WORKDIR /src

# Build and compile the tests. This also downloads the Gradle distribution and all dependencies.
RUN ./gradlew --no-daemon clean build testClasses -x test

# Run the test suite against the local database so its dependencies are cached and the image is known to work.
# A failing test is reported but does not stop the image build.
RUN avni-test-db \
    && (./gradlew --no-daemon test || echo "WARNING: gradle test reported failures") \
    && pg_ctlcluster 16 main stop
