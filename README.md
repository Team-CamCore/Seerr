<!-- CAMCORE-REPOSITORY-BRANDING:START -->
<p align="center">
  <a href="https://camcore.au">
    <img src=".github/brand/camcore-repository-banner.png" alt="CamCore" width="640">
  </a>
</p>
<!-- CAMCORE-REPOSITORY-BRANDING:END -->

<p align="center">
  <strong>Cameron-Media Requests</strong><br>
  CamCore-maintained deployment of Seerr for requesting movies and series on Cameron-Media.
</p>

<p align="center">
  <a href="https://requests.camcore.au"><strong>Open Cameron-Media Requests</strong></a>
  ·
  <a href="https://camcore.au/status">CamCore Status</a>
  ·
  <a href="https://camcore.au/help">Help centre</a>
  ·
  <a href="https://github.com/seerr-team/seerr">Upstream Seerr</a>
</p>

## About this repository

This repository is the CamCore-maintained fork of [Seerr](https://github.com/seerr-team/seerr). It preserves the upstream media-request platform while applying CamCore visual identity, operational defaults, email communication standards and a CamCore container build for Cameron-Media Requests.

The application continues to use Seerr's upstream request, discovery, user, notification and administration functionality, including Plex, Sonarr and Radarr integration.

CamCore-specific changes are intentionally kept narrow so upstream fixes can continue to be incorporated with minimal conflict.

## CamCore service identity

| Surface | CamCore identity |
| --- | --- |
| Service | Cameron-Media Requests |
| Public URL | `https://requests.camcore.au` |
| Container | `ghcr.io/team-camcore/seerr` |
| Email sender | `Requests | CamCore Media <help@camcore.au>` |
| Support | `https://camcore.au/support` |
| Help centre | `https://camcore.au/help` |
| CamCore Status | `https://camcore.au/status` |
| Time zone | `Australia/Melbourne` |

## CamCore customisation

The downstream layer includes:

- CamCore sign-in branding;
- CamCore desktop and mobile navigation;
- CamCore browser and PWA identity;
- CamCore offline page;
- links to CamCore home, Help centre, CamCore Status, Cameron-Media and Support;
- CamCore-branded request and account emails;
- CamCore GHCR image metadata and publishing workflow;
- CamCore deployment, verification and rollback guidance.

The application title stored in the existing Seerr configuration remains configurable and is not reset by the branded image.

### Where the CamCore changes live

The CamCore layer is a small set of files on top of upstream. These are the ones to check most carefully when merging upstream:

| Area | Files |
| --- | --- |
| Sign-in, navigation and page chrome | `src/components/Login/`, `src/components/Layout/`, `src/components/ResetPassword/`, `src/pages/_document.tsx`, `tailwind.config.js` |
| Browser, PWA and offline identity | `public/` logos and icons, `public/site.webmanifest`, `public/offline.html` |
| Email | `server/templates/email/_partials/camcore.pug`, each template's `html.pug` and `subject.pug`, `server/lib/email/index.ts`, `server/lib/notifications/agents/email.ts` |
| CI and deployment | `.github/workflows/camcore-image.yml`, `docs/camcore-deployment.md` |

CamCore also made small changes to two upstream workflows: `codeql.yml` excludes test files, and `trivy-scan.yml` signs in to GHCR so it can scan the private CamCore image.

## Email communication standard

Seerr email has been brought into the same communication system used by CamCore Operations support mail.

All Cameron-Media Requests email templates use a common shared layout with:

- the dark CamCore brand header;
- Core Coral brand divider;
- `CAMCORE MEDIA • REQUESTS` service labelling;
- event/status badges;
- structured light detail panels;
- Accessible Coral primary actions;
- CamCore Help centre, CamCore Status and support contact details;
- clear notification subjects without a redundant brand prefix.

Covered messages include request lifecycle updates, media issues, account creation, password reset and test notifications.

The expected sender identity is:

```text
Requests | CamCore Media <help@camcore.au>
```

The application remains **Cameron-Media Requests**. `CamCore Media` is used as the sender/brand family rather than as a replacement service name.

See [CamCore Deployment](./docs/camcore-deployment.md) for the SMTP and validation checklist.

## Container image

Successful CamCore builds publish:

```text
ghcr.io/team-camcore/seerr:latest
ghcr.io/team-camcore/seerr:camcore
ghcr.io/team-camcore/seerr:develop-<sha>
```

The CamCore production image is built for:

```text
linux/amd64
```

All three tags are published only after the CamCore validation job succeeds. `latest` and `camcore` move with every build; production is pinned to the immutable `develop-<sha>` tag, which is also the rollback target. Production deployments are made under an approved Change in YouTrack.

The package is private. Production runs on Neptune, which has no GHCR login of its own, so images are pulled through Portainer (OPS-1531). The running version, rollback image and host paths are recorded in the Knowledge Base article *CMA-A-20 Seerr and Media Requests*.

## CI

The CamCore workflow is `CamCore Seerr Image` ([`.github/workflows/camcore-image.yml`](.github/workflows/camcore-image.yml)). It runs on pushes to `develop` and `agent/**` branches, and by hand. Changes that only touch Markdown files or `docs/` do not start it.

- **Validate CamCore fork** runs on every push. It compiles the CamCore email templates, then runs `pnpm format:check`, `pnpm lint`, `pnpm typecheck`, `pnpm test` and `pnpm build`.
- **Build and publish AMD64 image** runs only on `develop`, after validation passes, and publishes the three tags above.

Both jobs use GitHub-hosted runners. The upstream `Trivy Container Vulnerability Scan` workflow scans `ghcr.io/team-camcore/seerr:latest` every Friday and uploads the results to code scanning (OPS-1507).

The other workflows in `.github/workflows` are upstream's and are kept unchanged to make upstream merges easier.

## Updating the existing service

Seerr stores persistent application data under:

```text
/app/config
```

Keep the existing host folder mapped to `/app/config` when moving to a new CamCore image. This preserves the database, users, requests, Plex connection, Sonarr and Radarr settings, notification agents and application preferences.

Back up the mapped config folder before an application update.

See [CamCore Deployment](./docs/camcore-deployment.md) for the full deployment, verification and rollback process.

## Example Docker Compose service

```yaml
services:
  seerr:
    image: ghcr.io/team-camcore/seerr:develop-<sha>
    init: true
    container_name: seerr
    restart: unless-stopped
    environment:
      - TZ=Australia/Melbourne
      - PORT=5055
    ports:
      - "5055:5055"
    volumes:
      - /path/to/existing/seerr/config:/app/config
    healthcheck:
      test: wget --no-verbose --tries=1 --spider http://localhost:5055/api/v1/settings/public || exit 1
      start_period: 20s
      timeout: 3s
      interval: 15s
      retries: 3
```

Replace `develop-<sha>` with the build being deployed, and the example host path with the exact config-folder path already used by the current container.

## Development

Install dependencies:

```sh
pnpm install --frozen-lockfile
```

Start the development environment:

```sh
pnpm dev
```

Run the CamCore validation checks:

```sh
pnpm format:check
pnpm lint
pnpm typecheck
pnpm test
```

Build the production application:

```sh
pnpm build
```

Build the container locally:

```sh
docker build -t camcore-seerr:local .
```

## Keeping the fork current

The CamCore fork should regularly incorporate security fixes and stable changes from `seerr-team/seerr`.

When updating from upstream:

1. review upstream release notes and migration guidance;
2. merge upstream `develop` into the CamCore fork with a merge commit, not a squash, so later syncs stay clean. The fork has no `upstream` remote, so fetch it directly: `git fetch https://github.com/seerr-team/seerr.git develop`, then `git merge FETCH_HEAD`;
3. resolve branding conflicts without removing upstream functionality;
4. pay particular attention to the small set of CamCore-modified UI and email files;
5. allow the CamCore validation workflow to complete;
6. publish the validated CamCore image;
7. deploy while retaining the existing `/app/config` mapping;
8. verify the live service, integrations and email delivery;
9. record material changes in CamCore Operations.

## Support and upstream issues

CamCore-specific deployment, branding, access and operational matters should use [CamCore Support](https://camcore.au/support).

For general Seerr application behaviour, first check the upstream project:

- [Seerr source repository](https://github.com/seerr-team/seerr)
- [Seerr documentation](https://docs.seerr.dev)
- [Seerr issue tracker](https://github.com/seerr-team/seerr/issues)
- [Seerr releases](https://github.com/seerr-team/seerr/releases)

## Credits

The underlying Seerr application is developed and maintained by the [Seerr team](https://github.com/seerr-team) and its open-source contributors.

CamCore maintains the branding, deployment workflow, email communication layer and operational documentation in this fork.

CamCore does not claim ownership of the original Seerr project.

## Licence

This repository remains subject to the upstream [MIT Licence](LICENSE).
