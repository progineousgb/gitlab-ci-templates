<p align="center"><img src="assets/logo.png" alt="Pro Gineous" width="360"></p>

# gitlab-ci-templates

GitLab CI includes for deploying to a plain Linux server over SSH. No agents, no Kubernetes, just `ssh` and `rsync` from an Alpine job image.

Each deploy template defines a hidden job (name starts with a dot). Include the file, then `extends:` the hidden job in your own pipeline and set the variables. Nothing runs until you do that, so including a template is safe. `lint.yml` is the exception: it adds two ready-to-run jobs.

| File | Hidden job | Use it for |
|------|-----------|------------|
| [templates/rsync-static.yml](templates/rsync-static.yml) | `.pg-rsync-static` | Built static sites (Hugo, Astro, Vite output) |
| [templates/compose-ssh.yml](templates/compose-ssh.yml) | `.pg-compose-ssh` | Apps run with `docker compose` on the server |
| [templates/laravel-deploy.yml](templates/laravel-deploy.yml) | `.pg-laravel-deploy` | Laravel with release directories and a `current` symlink |
| [templates/wordpress-deploy.yml](templates/wordpress-deploy.yml) | `.pg-wordpress-deploy` | One theme or plugin into `wp-content` |
| [templates/lint.yml](templates/lint.yml) | `shellcheck`, `yamllint` jobs | Linting shell scripts and YAML |

Full pipelines are in [examples/](examples/).

## Usage

```yaml
include:
  - project: pro-gineous-group/gitlab-ci-templates
    ref: main
    file: templates/rsync-static.yml

deploy:
  extends: .pg-rsync-static
  variables:
    STATIC_DIR: dist
  environment: production
```

Pin `ref` to a tag or commit SHA once you have something working, so a change here cannot break your deploys unannounced. If you cannot reach gitlab.com projects from your instance, use `include: remote:` with the raw file URL, or copy the file into your repo.

## Variables

Set these under Settings > CI/CD > Variables. Protect them so only protected branches and tags can use them.

### Used by every deploy template

| Variable | Type | Required | Notes |
|----------|------|----------|-------|
| `SSH_PRIVATE_KEY` | File | yes | Private key for the deploy user. Must be a File variable; the job runs `ssh-add "$SSH_PRIVATE_KEY"` on the file path. Make sure the value ends with a newline. |
| `SSH_KNOWN_HOSTS` | Variable | yes | Output of `ssh-keyscan -p 22 your.server`. Check the fingerprints against the server before saving. Host key checking stays on. |
| `DEPLOY_HOST` | Variable | yes | Hostname or IP. |
| `DEPLOY_USER` | Variable | yes | SSH user. Use a dedicated deploy user, not root. |
| `DEPLOY_PATH` | Variable | yes (not for WordPress) | Absolute path on the server. |
| `DEPLOY_PORT` | Variable | no | Default `22`. |

Generate a dedicated key for CI rather than reusing your own:

```bash
ssh-keygen -t ed25519 -N '' -C gitlab-deploy -f deploy_key
# deploy_key.pub goes into ~deploy/.ssh/authorized_keys on the server
# deploy_key goes into SSH_PRIVATE_KEY (type: File)
```

The jobs use `resource_group: $CI_ENVIRONMENT_NAME`, so two deploys to the same environment never run at once. Set `environment:` on your job for that to take effect.

### rsync-static.yml

| Variable | Default | Notes |
|----------|---------|-------|
| `STATIC_DIR` | `public` | Directory to upload. Must come from an earlier job's `artifacts`. |
| `RSYNC_EXTRA_ARGS` | empty | For example `--exclude .well-known`. |

`--delete` is on: files on the server that are not in `STATIC_DIR` are removed. Point `DEPLOY_PATH` at the document root, not its parent.

### compose-ssh.yml

| Variable | Default | Notes |
|----------|---------|-------|
| `COMPOSE_FILE_NAME` | `compose.yaml` | Relative to `DEPLOY_PATH`. |
| `COMPOSE_REGISTRY_LOGIN` | `false` | Set `true` to log the server into `$CI_REGISTRY` before pulling. |
| `REGISTRY_USER`, `REGISTRY_PASSWORD` | none | Needed when `COMPOSE_REGISTRY_LOGIN=true`. Use a deploy token with `read_registry`. `CI_JOB_TOKEN` will not work because it expires when the job ends and the server keeps the login. |

The repository is synced to `DEPLOY_PATH` without `.git` and `.env`. Keep the real `.env` on the server. The deploy user needs to be in the `docker` group, which is root-equivalent; treat that key accordingly.

### laravel-deploy.yml

| Variable | Default | Notes |
|----------|---------|-------|
| `PHP_BIN` | `php` | For example `php8.3` on servers with several versions. |
| `COMPOSER_BIN` | `composer` | Composer must be installed on the server. |
| `RUN_MIGRATIONS` | `true` | Runs `artisan migrate --force`. |
| `KEEP_RELEASES` | `5` | How many releases to keep. Releases are named `<pipeline IID>-<short sha>` and ordered by the pipeline IID, not by file time. Older ones are deleted; the release `current` points to is always kept. |
| `PHP_FPM_RELOAD` | empty | Command to reload PHP-FPM so OPcache picks up the new path, e.g. `sudo systemctl reload php8.3-fpm`. Needs a matching sudoers rule. |

Before the first deploy, create `$DEPLOY_PATH/shared/.env` by hand. The job fails if it is missing. Point the web server at `$DEPLOY_PATH/current/public`. Deploy variables that end up on the server (`DEPLOY_PATH`, `WP_PATH`, and similar) must not contain a single quote; the job stops if one does.

Frontend assets: build them in an earlier job and pass `public/build` as an artifact. The deploy job uploads whatever is in the working tree.

Rolling back is a symlink change:

```bash
ln -sfn /var/www/app/releases/<old-release> /var/www/app/current
```

Migrations are not rolled back by that, so write them to be backwards compatible.

### wordpress-deploy.yml

| Variable | Default | Notes |
|----------|---------|-------|
| `WP_PATH` | none | WordPress root (where `wp-config.php` is). |
| `WP_TYPE` | none | `themes` or `plugins`. |
| `WP_SLUG` | none | Directory name under `wp-content/<type>/`. |
| `WP_SOURCE_DIR` | `.` | Subdirectory of the repo to upload. |
| `WP_FLUSH_CACHE` | `false` | Runs `wp cache flush`; needs WP-CLI on the server. |

The deploy user needs write access to the target directory only, not the whole WordPress install.

### lint.yml

| Variable | Default | Notes |
|----------|---------|-------|
| `SHELLCHECK_OPTS` | empty | Extra flags, e.g. `-S warning`. |
| `YAMLLINT_CONFIG` | relaxed default | Used only when the repo has no `.yamllint`, `.yamllint.yml` or `.yamllint.yaml`. |

`shellcheck` checks tracked files that end in `.sh`/`.bash` or start with a sh/bash shebang. Both jobs need a `test` stage in your pipeline.

## Validation

All YAML in this repo passes `yamllint -s .`. The job scripts were extracted and checked with `sh -n` and `shellcheck -s sh`. The templates have not been run against a live GitLab instance from this repository's CI, so run `CI Lint` (Build > Pipeline editor) on your pipeline after including them.

## License

MIT, see [LICENSE](LICENSE).

---

Built by the team at [Pro Gineous](https://progineous.com/). Related shell scripts: [server-toolkit](https://gitlab.com/pro-gineous-group/server-toolkit).
