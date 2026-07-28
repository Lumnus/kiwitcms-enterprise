# Lumnus in-house build of Kiwi TCMS Enterprise

This fork exists so the Lumnus substrate can **build the Kiwi TCMS Enterprise Edition
in-house from public sources**, without the subscriber-only registry or the private
`PKG_TOKEN` package index.

- **License:** AGPL-3.0+ (unchanged from upstream `kiwitcms/enterprise`). Our fork is public
  per the substrate's fork-visibility discipline; §13 network-copyleft is satisfied by that.
- **Build:** [`lumnus/Dockerfile`](lumnus/Dockerfile) — a multi-stage self-build that
  (1) builds the `kiwitcms_enterprise` wheel from THIS source, (2) bases on the **public** CE
  image `pub.kiwitcms.eu/kiwitcms/kiwi` (16.2, matches enterprise HEAD), (3) resolves every
  dep from public PyPI. Upstream `Dockerfile` is left untouched (subscriber path) for clean diffs.
- **Delivers** (over CE): Keycloak/OIDC SSO · multi-tenancy (`kiwitcms-tenants`) · psycopg-pool ·
  django-prometheus · S3/MinIO uploads (`django-storages`).
- **Built via** the substrate's `ci-build` → Zot; wired as a profile in the Conductor's
  `imageRebuild` buildflow family. Deployed at `infra/cluster-blast/apps/kiwi/`.

Deep gratitude to **Alexander Todorov** and the Kiwi TCMS team for releasing the Enterprise
Edition under AGPL — genuinely generous. We intend to contribute back (a Plane tracker backend).
