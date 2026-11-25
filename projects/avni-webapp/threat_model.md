# Threat model

The scanner reads this file before it starts. It covers avni-webapp.

## What this project does and where untrusted input enters

Avni is an open-source platform that NGOs and governments use for field work in health, nutrition, water, education
and social welfare. Field workers collect data on an offline-first Android app that syncs with a central server. The
server stores personal and health data about the people those organisations serve, and one deployment holds many
organisations (tenants). Reading or changing another organisation's data is the most serious class of finding in
every Avni component.

avni-webapp is Avni's browser application (React, built with Vite). It contains the administration screens (users,
catchments, locations), the app designer (forms, concepts, rules, report cards, dashboards), bulk upload, web data
entry, the news and documentation editors, and the help assistant. It is served as static files from the same origin
as avni-server and talks to it with a Cognito or Keycloak token kept in the browser. It has no server-side code; a
finding here is something that runs in, or misleads, another user's browser session.

Untrusted input reaches it through:

- **The URL**: path, query string and hash, including values used for navigation after sign-in.
- **Content authored by other users and returned by the server**: names and descriptions, news and documentation
  HTML and markdown, extension and custom-card HTML, translations, uploaded files that are parsed in the browser
  (zip bundles, CSV, spreadsheets), and observation values typed by field users.
- **Rule code written by an organisation administrator**, which the data-entry screens evaluate in the browser of
  whoever opens the form. An administrator of one organisation must not be able to run code in the session of a user
  of another organisation or of an Avni operator.
- **Responses from third-party services** the app embeds or calls.

### Trust model

- Organisation administrators are trusted only inside their own organisation; an organisation can be created by
  self-signup. Field and web users are less trusted than administrators. Avni operators (super administrators) use
  the same app across organisations and are the most valuable target. The server enforces authorisation; hiding a
  button in the UI is not a security control, so a missing server check belongs to avni-server, not here.

## Components that matter most / least

### Highest priority

1. Sign-in, token storage and sign-out (`src/rootApp/`, the Cognito and Keycloak helpers), and any redirect or
   navigation built from the URL.
2. Every place HTML, markdown or a URL from stored content is rendered or opened (news, documentation, extensions,
   custom cards, links, iframes).
3. Evaluation of rule code and other strings as code in the data-entry app (`src/dataEntryApp/`,
   `src/formDesigner/`).
4. Files parsed in the browser and what is done with their contents.
5. Tokens placed in URLs, logs or requests to other origins.

### Lower priority and out of scope

- Out of scope: tests, `cypress/`, `__mocks__/`, `scripts/`, `translations/` JSON, the root `Dockerfile`, build
  and CI configuration, and findings that only affect the development server.

## How to exercise it

- `/src` is the checkout; the image ran `yarn install`, `yarn run build` (output in `build/`) and `yarn test` on
  Node 24.4.1.
- Run the tests with `yarn test` (Jest with Testing Library). A proof of concept is a Jest test that renders the
  component with hostile content, or the exact URL or stored value and the component and line that handle it.
  avni-server is not available in the image.

## How you rate severity

- **Critical**: script execution in another organisation's user's session or in an Avni operator's session, or
  theft of their token, without that user doing more than opening a link or a normal screen.
- **High**: the same inside one organisation by a user who is not its administrator; open navigation to a
  script-capable URL after sign-in.
- **Medium**: open redirect; content spoofing; tokens exposed in URLs or to third parties; a session that survives
  sign-out.
- **Low**: self-XSS; issues that need an administrator to attack their own organisation; hardening gaps with no
  demonstrated outcome.

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
