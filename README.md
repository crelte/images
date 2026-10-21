# images

Container images used by crelte projects, published on Docker Hub.

## craft

[`crelte/craft`](https://hub.docker.com/r/crelte/craft) is a drop-in
replacement for the archived `ghcr.io/craftcms/image`: same Ubuntu 24.04 base,
PHP from `ppa:ondrej/php`, packages, paths, `appuser` (3000:3000), `PHP_*`
environment variables and supervisord entrypoint. To switch a project, only
change the `FROM` line:

```Dockerfile
FROM crelte/craft:8.4
```

Tags: `8.4` (moving) and the exact PHP version, e.g. `8.4.27`.

Differences to the craftcms image:

- the build fails if the PPA delivers PHP older than `php_min_version`
- the Craft server check runs from a pinned release with a checksum
  instead of piping `HEAD/check.sh` into bash
- apt retries downloads 10 times

### Build locally

```bash
docker build --pull -t crelte/craft:8.4 craft
```

### CI

`.github/workflows/craft.yml` builds every day at 04:00 UTC and pushes
`linux/amd64` + `linux/arm64` when the PHP version changed, every Monday
(Ubuntu updates), on a push to `main` and when started by hand. The Docker
Hub credentials are the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets of
the `Main Deployments` environment.

When a PHP security release comes out, raise `min` in the workflow matrix
so the image can't be pushed with an older version.
