# images

Container images used by crelte projects, published on Docker Hub.

## craft

[`crelte/craft`](https://hub.docker.com/r/crelte/craft) is a drop-in
replacement for the archived `ghcr.io/craftcms/image`: same Ubuntu 24.04 base,
packages, paths, `appuser` (3000:3000), `PHP_*`
environment variables and supervisord entrypoint. To switch a project, only
change the `FROM` line:

```Dockerfile
FROM crelte/craft:8.4
```

Tags: `8.4` (moving) and the exact PHP version, e.g. `8.4.27`.

Differences to the craftcms image:

- PHP comes from `packages.sury.org` instead of `ppa:ondrej/php` (same
  packages by the same maintainer; the PPA is being merged into it)
- the build fails if the repo delivers PHP older than `php_min_version`
- the Craft server check runs from a pinned 5.x release with a checksum
  instead of piping `HEAD/check.sh` into bash (6.x requires PHP 8.5)
- `libheif-plugin-libde265` is installed explicitly, newer libheif no
  longer pulls it in
- apt retries downloads 10 times
- only `linux/amd64` until sury.org ships the fixed PHP for arm64

### Build locally

```bash
docker build --pull -t crelte/craft:8.4 craft
```

### CI

`.github/workflows/craft.yml` builds every day at 04:00 UTC and pushes
when the PHP version changed, every Monday
(Ubuntu updates), on a push to `main` and when started by hand. The Docker
Hub credentials are the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets of
the `Main Deployments` environment.

When a PHP security release comes out, raise `min` in the workflow matrix
so the image can't be pushed with an older version.
