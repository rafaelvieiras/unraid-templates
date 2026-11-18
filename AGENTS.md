# AGENTS.md — unraid-templates

Rules for any agent (or person) editing a template in this repository. Read this before
touching `templates/*.xml`. Everything here is in English, like the rest of the repo.

## What this repository is

CA (Community Applications) is the Unraid app store. It hosts **nothing**: it reads a
maintainer profile plus one XML template per container, over raw GitHub URLs, and lists
the apps after review. So a template is not documentation — it is the **script that builds
the container** on somebody else's server. A wrong element does not print a warning: it
produces a container that starts and misbehaves.

## Rule 1 — only modern `<Config>` entries. Never the legacy blocks.

**Never add `<Networking>`, `<Data>` or `<Environment>`.** Ports, paths and variables go
in `<Config>` entries (`Type="Port"`, `Type="Path"`, `Type="Variable"`) and nowhere else.

Why this is a hard rule, not a style preference: on 17/nov/2026 a container was recreated
by the Unraid auto-update from a template that carried **both** the legacy blocks and the
modern `<Config>` entries. It came up with:

- an **empty environment** (the TMDB key and the paths were gone), and
- `PortBindings=map[8080/tcp:[{invalid IP 8080}]]` — no usable port at all.

The app was healthy; the container was unusable, and it stayed that way for six hours
because the recreate happened at 04:00. The blocks duplicate what the `<Config>` entries
already declare, and when both exist the merge is not something to rely on.

Check before every commit:

```bash
grep -nE "<(Networking|Data|Environment)>" templates/*.xml   # must print nothing
```

If a value only exists in a legacy block, it is **missing** from the template: move it into
a `<Config>` entry, do not delete it. A functional example: `m3m0s.xml` needs
`M3M0S_DATA_DIR=/data` as a `<Config>`, because the image default outside the volume is
`~/.m3m0s/data`.

## Rule 2 — the image tag carries the releases

`<Repository>` points at `:latest` whenever the project publishes it. Users then receive
releases through the image, and the template is edited only when the **configuration
surface** changes (new env var, port, volume, description).

Pin a version only when the project deliberately does not publish `latest` —
`playheadtv.xml` is the case: its releases are `v*` and `client-v*`, so the template pins
`v0.3.15`. Do not invent a tag that the registry does not have.

## Rule 3 — permissions through the image's own contract

Default Unraid ids are `PUID=99` / `PGID=100` (`nobody:users`); without them the app cannot
write to `/mnt/user/appdata`.

- If the image honours `PUID`/`PGID` (MyMediaDB does, since 0.5.1), expose them as
  `<Config>` variables with those defaults and **do not** add `--user` to `<ExtraParams>` —
  a `--user` flag prevents the entrypoint from fixing the ownership at all.
- If the image does not honour them, use `<ExtraParams>--user 99:100</ExtraParams>`.

## Rule 4 — a recreate must not lose data or settings

Every value the app needs to come back identical belongs in the template, because a
recreate reads the template and nothing else:

- one `<Config Type="Path">` per mount, with the host path as `Default`;
- one `<Config Type="Port">` per port, and `<WebUI>` refers to the **container** port (Unraid
  rewrites it when the user remaps the host port);
- one `<Config Type="Variable" Mask="true">` per secret, so it is not echoed in the UI;
- `<Category>` tokens each end in a colon (`MediaApp:Other: Tools:Utilities:`);
- `<Date>` and `<Changes>` updated: `<Changes>` is what the user sees before upgrading, and
  it is where a behaviour change gets explained in one line.

## Rule 5 — validate before committing

```bash
xmllint --noout ca_profile.xml templates/*.xml
grep -nE "<(Networking|Data|Environment)>" templates/*.xml      # must be empty
grep -o '<Config Name="[^"]*"' templates/<app>.xml               # the surface, at a glance
```

Then confirm the numbers in the template against the image itself, not against memory:
`docker inspect <image>` for `ExposedPorts`, `Volumes` and `Env`. A template that declares a
port the image does not expose builds a container that never answers.

## Rule 6 — `templates/_skeleton.xml.example` is the reference

It already uses only `<Config>` entries and carries `PUID`/`PGID`/`TZ`. Copy it when adding
an app. Keep the `.example` suffix: the CA scanner must never parse it.

## Repository layout

| Path | Purpose |
|------|---------|
| `ca_profile.xml` | Maintainer profile shown in CA. Must stay at the root. |
| `templates/<app>.xml` | One template per Docker app. |
| `templates/_skeleton.xml.example` | Skeleton, never scanned. |
| `icons/` | Square icons for apps whose own repository only ships a wide logo. |
| `icon.svg` | Repository icon referenced by `ca_profile.xml`. |

## Before opening a PR

- [ ] `xmllint` clean, legacy grep empty.
- [ ] `<Repository>`, `<Registry>`, `<Project>`, `<Support>`, `<TemplateURL>` and `<Icon>`
      URLs resolve (open them).
- [ ] Ports/volumes/variables match the image (`docker inspect`).
- [ ] `<Date>` and `<Changes>` updated.
- [ ] `README.md` app table updated when the image tag, port or permission model changed.
