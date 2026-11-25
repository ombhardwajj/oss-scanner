# Threat model

The scanner reads this file before it starts. It covers avni-media.

## What this project does and where untrusted input enters

Avni is an open-source platform that NGOs and governments use for field work in health, nutrition, water, education
and social welfare. Field workers collect data on an offline-first Android app that syncs with a central server. The
server stores personal and health data about the people those organisations serve, and one deployment holds many
organisations (tenants). Reading or changing another organisation's data is the most serious class of finding in
every Avni component.

avni-media is the media viewer for Avni: a Next.js web client (`client/`) in which an organisation's users browse,
filter and select the photos, videos and documents their field workers captured, and a NestJS server (`server/`)
that prepares bulk downloads. The server stores download requests in PostgreSQL, and a scheduled task fetches the
selected objects from S3-compatible storage with the service's own credentials, zips them, uploads the zip and hands
back a pre-signed link. All organisations' media live in one bucket under per-organisation prefixes, so which keys a
caller can make the server read is the central question.

Untrusted input reaches it through:

- **The server's HTTP API** (`server/src/media-viewer/`): request bodies and query parameters, including media URLs,
  folder and file names, a user name and filter descriptions. Treat every caller as untrusted until the server has
  verified who they are and which organisation they belong to.
- **The web client**: the Avni token it holds in the browser, URL parameters, and media metadata (names, addresses,
  concept values) written by field users and rendered for other users.
- **Stored download requests**, processed later by the scheduled task.
- **Objects fetched from storage**, written by field users' phones.

### Trust model

- Unauthenticated clients are untrusted. Users are trusted only inside their own organisation. Avni's operators and
  the environment files are trusted.

## Components that matter most / least

### Highest priority

1. Authentication and organisation scoping of every server route, and of the storage keys a request can name.
2. File and path handling while building zips (`server/src/utils/file-utility.ts`, `media-viewer.service.ts`).
3. Database access through TypeORM with request-shaped objects.
4. The scheduled task: what one bad request does to the other requests in the queue, and resource limits.
5. The client's token handling, middleware and rendering of user-supplied text.

### Lower priority and out of scope

- Out of scope: tests and specs, `deploy/`, `scripts/`, environment templates, CI configuration.

## How to exercise it

- `/src` is the checkout. The image ran `npm install --legacy-peer-deps` and a build in `server/` and in
  `client/` on Node 19.8.1, the version the project builds with. Node 19.8.1 is in `/usr/local`.
- Server: `node dist/main` starts it on port 3010. `cd server && npm test` runs Jest, but the project's CI does not
  run these specs and the four existing ones do not compile against the current sources; write new specs rather than
  relying on them. It needs PostgreSQL and
  S3-compatible storage, which are not in the image, so stub them or show the issue with a unit test of the service
  or utility involved.
- Client: `cd client && npm test`.

## How you rate severity

- **Critical**: reading or downloading any organisation's media without authentication, or one organisation's user
  reaching another organisation's media; code execution or arbitrary file write on the server host.
- **High**: reading files outside the working directory; reading the service's storage or database credentials;
  one user reading another user's downloads inside the same organisation.
- **Medium**: stored XSS in the client; one request stalling or breaking the download queue for all; tokens in logs
  or URLs.
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
