# Deploying Paperless-ngx via Nix

Paperless-ngx is officially packaged in nixpkgs as a NixOS service module at `nixos/modules/services/misc/paperless.nix`. The project itself has no `.nix` files — the packaging lives entirely in nixpkgs. It runs as **native systemd services** (no Docker needed).

## Minimal Setup

```nix
environment.etc."paperless-admin-pass".text = "changeme";
services.paperless = {
  enable = true;
  passwordFile = "/etc/paperless-admin-pass";
};
```

This gives you Paperless at `http://localhost:28981` with Redis auto-configured, using SQLite.

## Recommended Production Setup

```nix
services.paperless = {
  enable = true;
  passwordFile = "/run/keys/paperless-password";  # or sops-nix, agenix, etc.
  database.createLocally = true;        # auto-configures PostgreSQL
  consumptionDirIsPublic = true;         # allows uploading via consume dir
  openMPThreadingWorkaround = true;      # default; prevents classifier hangs
  settings = {
    PAPERLESS_URL = "https://paperless.example.com";
    PAPERLESS_OCR_LANGUAGE = "deu+eng";
    PAPERLESS_CONSUMER_IGNORE_PATTERN = [ ".DS_STORE/*" "desktop.ini" ];
    PAPERLESS_OCR_USER_ARGS = {
      optimize = 1;
      pdfa_image_compression = "lossless";
    };
  };
};
services.paperless.configureNginx = true;   # auto-configures nginx reverse proxy
services.paperless.domain = "paperless.example.com";
```

## What the Module Manages

The NixOS module creates **5 systemd services** in a shared `system-paperless` slice:

| Service                | Purpose                                              |
| ---------------------- | ---------------------------------------------------- |
| `paperless-scheduler`  | Celery Beat + auto-migration + superuser setup       |
| `paperless-task-queue` | Celery workers (document processing, classification) |
| `paperless-consumer`   | Filesystem watcher for the consume directory         |
| `paperless-web`        | Granian ASGI server (web UI + WebSocket)             |
| `paperless-exporter`   | (Optional) Scheduled document export via timer       |

It also provides:

- **`paperless-manage`** — wrapper script for `manage.py` (auto-sudo to the `paperless` user)
- **Redis** — auto-provisioned with Unix socket, unless you set `PAPERLESS_REDIS` yourself
- **PostgreSQL** — optional via `database.createLocally = true`
- **Secret key** — auto-generated at `${dataDir}/nixos-paperless-secret-key`
- **Tesseract** — auto-configured with the languages you set in `PAPERLESS_OCR_LANGUAGE`

## Office/Email Support (Tika + Gotenberg)

The module has a toggle that configures **native NixOS services** (no containers needed):

```nix
services.paperless.configureTika = true;
```

This automatically enables `services.tika` and `services.gotenberg` and sets the corresponding `PAPERLESS_TIKA_*` and `PAPERLESS_GOTENBERG_*` environment variables.

## Secrets

Use `environmentFile` for secrets you don't want in the Nix store:

```nix
services.paperless = {
  environmentFile = "/run/secrets/paperless-env";  # sops-nix, agenix, etc.
};
# /run/secrets/paperless-env contains:
# PAPERLESS_DBPASS=s3cret
```

## Automated Exports

```nix
services.paperless.exporter = {
  enable = true;
  directory = "/var/lib/paperless/export";
  onCalendar = "01:30:00";  # daily at 1:30am (systemd timer calendar format)
};
```

## Key Gotchas

1. **`openMPThreadingWorkaround`** — defaults to `true` and you almost certainly want it. Without it, the scikit-learn classifier can spin indefinitely ([nixpkgs#240591](https://github.com/NixOS/nixpkgs/issues/240591)).
2. **Database hardening** — The systemd services use `PrivateNetwork = true` by default for security. If you use a remote database, the module disables this automatically, but be aware if you need network access for other reasons.
3. **Tesseract languages** — The package derivation auto-overrides tesseract to include only the languages specified in `PAPERLESS_OCR_LANGUAGE` plus `eng`, `equ`, and `osd`. If you don't set it, all languages are built.
4. **Version upgrades** — The scheduler service auto-detects package version changes and runs Django migrations automatically.
5. **Secret key** — Auto-generated on first start. Don't delete `${dataDir}/nixos-paperless-secret-key`.

## Non-NixOS Options

If you're not on NixOS but use Nix:

- **`nix-shell` / `flake.nix`** — There's no official flake, but you can use `pkgs.paperless-ngx` from nixpkgs as a package and write your own process supervisor.
- **Docker Compose with Nix** — Generate your `docker-compose.yml` from Nix expressions using `virtualisation.oci-containers` on NixOS or tools like `arion`.

## References

- NixOS Wiki: <https://wiki.nixos.org/wiki/Paperless-ngx>
- Module options: <https://mynixos.com/options/services.paperless>
- Module source: `nixos/modules/services/misc/paperless.nix` in [nixpkgs](https://github.com/NixOS/nixpkgs)
- Paperless-ngx docs: <https://docs.paperless-ngx.com/configuration/>
