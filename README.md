# oauth-client-metadata

OAuth 2.0 [Client ID Metadata Documents][cimd] served over GitHub Pages, for local development.

Under CIMD the `client_id` **is** the HTTPS URL the document is served from, so there is nothing to
register with an authorization server first. The `client_id` inside each document has to equal the URL it
is fetched from exactly — that binding is what stops one client impersonating another.

| Document | `client_id` |
|---|---|
| [`slack-app-server/client.json`](slack-app-server/client.json) | `https://rahulsingh-sumo.github.io/oauth-client-metadata/slack-app-server/client.json` |

**Public on purpose, and free of secrets.** These are public-client documents:
`token_endpoint_auth_method` is `none`, so there is no client secret to leak, and the authorization
server has to be able to fetch them anonymously. Nothing here should ever gain a credential.

## Constraints these documents are written to

- **No port on the loopback redirect URIs.** Loopback redirects are compared port-agnostically
  (RFC 8252 §7.3), so one document keeps working whatever port the local listener lands on. The *path* is
  compared exactly.
- **Served with a plain `200` and no redirect.** Sumo's `CimdMetadataHttpClient` calls
  `disableRedirectHandling()` and treats anything but 200 as failure. Pages serves these directly, which
  is why they are here rather than in a gist — a gist `/raw/` URL redirects to a revision-SHA URL that
  changes on every edit, and the `client_id` inside would stop matching.
- **Under 5 KB** (`oauth.cimd.metadata.client.max.response.bytes`).
- **`.nojekyll` at the root** so Pages publishes the tree verbatim instead of running it through Jekyll.

## Verifying a document after editing it

```bash
curl -si https://rahulsingh-sumo.github.io/oauth-client-metadata/slack-app-server/client.json | head -5
```

Must be `200` with **no** `Location:` header.

Sumo caches a fetched document for 30 minutes (`oauth.cimd.metadata.client.expire.minutes`), so an edit
is not picked up immediately. To force a new cache key, change the filename — and remember to change the
`client_id` inside to match.

## Using it

```bash
./gradlew :slack-app-server:runLocal \
  -Dslack.app.server.sumo.api.base.url=https://service.stag.sumologic.net \
  -Dslack.app.server.sumo.oauth.client.id=https://rahulsingh-sumo.github.io/oauth-client-metadata/slack-app-server/client.json \
  ...
```

A URL client id makes `SlackAppServerOAuthConfig.sumoClientIsPublic` true, which sends PKCE and no client
authentication to the token endpoint — hence no `sumo-client-secret`.

Only below-prod authorization servers advertise `client_id_metadata_document_supported: true`
(`service.stag.sumologic.net`, `service.long.sumologic.net`). Prod defaults `oAuthCimdPolicy` to
`disabled` and uses a registered confidential client instead.

Full runbook: `slack-app-server/run-readme.md` in the `sumologic` repo.

[cimd]: https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/
