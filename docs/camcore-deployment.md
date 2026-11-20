# CamCore Deployment

This fork publishes the CamCore-branded Cameron-Media Requests image to:

```text
ghcr.io/team-camcore/seerr
```

The CamCore production image is built for `linux/amd64`.

Production runs on Neptune and is **pinned to an immutable `develop-<sha>` tag**, not `latest`. Every production deployment needs an approved Change in YouTrack (OPS project) before it starts. The current running version, rollback image and host paths are recorded in the Knowledge Base article *CMA-A-20 Seerr and Media Requests*.

## Release model

A push to `develop` does not publish the CamCore image until the fork has passed its validation job. The CamCore workflow runs formatting, linting, type checking and unit tests before the AMD64 container is built and pushed.

Published tags include:

```text
ghcr.io/team-camcore/seerr:latest
ghcr.io/team-camcore/seerr:camcore
ghcr.io/team-camcore/seerr:develop-<sha>
```

Deploy the `develop-<sha>` tag. It never changes, so the running version only moves when the Compose file is edited under a Change, and it is the rollback target for that build. `latest` and `camcore` move on every successful build to `develop`; do not run production on them.

## Preserve the existing configuration

Seerr stores its persistent database, settings, users, requests, integrations and notification configuration under:

```text
/app/config
```

When replacing the current image, keep the existing host folder mapped to `/app/config`. Do not create a new config folder and do not remove the current volume mapping.

Only the container image needs to change.

## Before updating

1. Confirm the Change for this deployment is approved and record its evidence there.
2. Confirm the CamCore image workflow completed successfully for the version being deployed.
3. Record the running image and its ID (the rollback point):

   ```sh
   docker inspect seerr --format 'image={{.Config.Image}} id={{.Image}}'
   ```

4. Confirm the Compose file still describes the running container. Its `image:` line must match the `image=` value above. If it does not, stop and record the drift in the Change before going further.
5. Record the current port mapping, environment variables, network and reverse-proxy destination.
6. Back up the existing host folder that is mapped to `/app/config`.
7. Confirm that the existing config folder is writable by UID and GID `1000:1000`.

Example permission repair when required:

```sh
docker run --rm \
  -v /path/to/existing/seerr/config:/data \
  alpine chown -R 1000:1000 /data
```

## Docker Compose

Keep the current service definition and change only the `image` value when moving between CamCore builds. Use the immutable tag of the build being deployed.

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

Replace `/path/to/existing/seerr/config` with the exact host path already used by the current container.

Keep a copy of the Compose file before editing it, then pull the image and recreate only the `seerr` service:

```sh
docker pull ghcr.io/team-camcore/seerr:develop-<sha>
docker compose up -d --pull never seerr
```

`--pull never` makes Compose use the image already pulled, so the recreate cannot fetch a different build.

## Neptune (production)

The production container runs on Neptune as the Compose project `seerr`. The Compose file path, config folder path and current image are recorded in *CMA-A-20 Seerr and Media Requests*.

The `ghcr.io/team-camcore/seerr` package is private and, as at 20 November 2026, Neptune has no GHCR login, so `docker pull` on the host fails with `unauthorized`. Pull the image through Portainer instead (Neptune environment → Images, `ghcr.io` registry), then confirm its digest on the host:

```sh
docker image inspect ghcr.io/team-camcore/seerr:develop-<sha> --format '{{index .RepoDigests 0}}'
```

Then recreate with `docker compose up -d --pull never seerr` from the Compose project folder. Change the container through its Compose file, not through Portainer's recreate, so the file keeps describing what is running. The pull credential is tracked under OPS-1531.

## Synology Container Manager

When updating through Synology Container Manager:

1. Confirm the existing container's volume mapping to `/app/config`.
2. Back up the mapped host folder.
3. Change the image to the required `ghcr.io/team-camcore/seerr` tag.
4. Keep the existing container name, ports, environment variables, network and volume mappings.
5. Recreate or update the container.
6. Confirm that the application opens and existing requests, users and integrations are present.

If the GitHub Container Registry package is private, authenticate Container Manager to `ghcr.io`. Otherwise keep the package public for pull-only deployment.

## CamCore email identity

Cameron-Media Requests uses the CamCore application sender standard:

| Setting | Value |
| --- | --- |
| Sender name | `Requests | CamCore Media` |
| From address | `help@camcore.au` |
| Full From identity | `Requests \| CamCore Media <help@camcore.au>` |
| SMTP host | `192.168.5.101` |
| SMTP port | `25` |
| Encryption | None on the private CamCore LAN |
| Use | Request confirmations and status updates |

The application/service name remains **Cameron-Media Requests**. `CamCore Media` is used as the sender and brand family, not as a replacement application name.

The CamCore fork also provides branded HTML templates for:

- request pending, approved, available, declined and failed notifications;
- automatically submitted and automatically approved requests;
- media issue notifications and comments;
- account creation;
- password reset;
- email test notifications.

The email design follows the same communication system used by CamCore Operations support messages: a dark CamCore header, Core Coral divider, service label, event badge, white content area, structured detail panels, Accessible Coral primary action and CamCore footer links.

After changing email settings, send a test notification and confirm:

- the sender is `Requests | CamCore Media <help@camcore.au>`;
- the CamCore logo loads;
- the test subject is `Test notification — Cameron-Media Requests` with no `[CamCore Media]` prefix;
- the email displays correctly in Outlook desktop and Outlook on the web;
- the Help centre and CamCore Status links work;
- request and issue buttons return to `https://requests.camcore.au`.

## Post-deployment verification

After deployment, confirm:

- the CamCore logo appears on the sign-in page and navigation;
- existing user accounts remain available;
- existing requests and request history remain available;
- Plex authentication works;
- Sonarr and Radarr connections test successfully;
- notification agents remain configured;
- a CamCore-branded test email is received successfully;
- `https://requests.camcore.au` loads through the existing reverse proxy;
- the health endpoint responds successfully:

```text
/api/v1/settings/public
```

## Rollback

Roll back to the immutable tag recorded as the rollback point in the Change:

```text
ghcr.io/team-camcore/seerr:develop-<previous sha>
```

To roll back without losing configuration:

1. Set the Compose `image:` line back to the previous `develop-<sha>` tag. Keep the same `/app/config` volume mapping.
2. If that image is no longer on the host, pull it first (through Portainer on Neptune).
3. Recreate the container with `docker compose up -d --pull never seerr`.
4. Verify requests, integrations and email notifications.
5. Record the rollback in the Change.

If a database migration prevents an image-only rollback, stop the container and restore the config-folder backup taken before the update.

## Upstream updates

CamCore branding changes should remain separate from upstream functionality where practical. When pulling new changes from `seerr-team/seerr`:

1. merge upstream into the CamCore fork;
2. review conflicts in the CamCore branding and email template files carefully;
3. allow validation to complete before publishing/deploying the resulting image;
4. deploy the resulting `develop-<sha>` image under a Change and verify the live service;
5. use a merge commit, not a squash, so later upstream syncs stay clean.
