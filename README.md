# unraid-templates

Unraid Community Applications (CA) templates for my self-hosted projects.

CA does not host anything. This repository is the metadata layer: a maintainer profile
plus one XML template per container. The Unraid scanner reads the default branch over
raw GitHub URLs and lists the apps in the Apps tab after review.

## Layout

| Path | Purpose |
|------|---------|
| `ca_profile.xml` | Repository profile shown in CA. Must stay at the repository root. |
| `templates/` | One XML file per Docker app (`m3m0s.xml`, `tinyfin.xml`, `tinyfin-api.xml`, `my-media-db.xml`, `playheadtv.xml`, plus the `.example` skeleton). |
| `icons/` | Square **PNG** icons for apps whose own repository only ships a wide logo. The `.svg` next to each PNG is the source; it is never referenced by a template. |
| `plugins/` | One XML wrapper per Unraid plugin (none yet). |
| `icon.png` | Repository icon referenced by `ca_profile.xml` (`icon.svg` is the source). |

### Icons must be PNG

Unraid's dockerMan fetches `<Icon>` and caches it as `<Container>-icon.png`. When the URL
points to an SVG, the SVG bytes are saved under a `.png` name and the browser cannot decode
them: the container shows the generic placeholder. CA has the same limitation for the
profile icon. So every `<Icon>` and the `<Icon>` in `ca_profile.xml` point to a PNG.

| File | Referenced by |
|------|---------------|
| `icon.png` | `ca_profile.xml` |
| `icons/my-media-db.png` | `templates/my-media-db.xml` |
| `icons/tinyfin.png` | `templates/tinyfin.xml`, `templates/tinyfin-api.xml` |
| `icons/playheadtv.png` | `templates/playheadtv.xml` |

## Apps

| App | Image | Port | Volume | Notes |
|-----|-------|------|--------|-------|
| M3m0s | `ghcr.io/rafaelvieiras/m3m0s:latest` | 19234 | `/data` | Long-term memory for LLM agents. Runs as `99:100`. Public GHCR package: no registry login needed. || Tinyfin | `ghcr.io/rafaelvieiras/tinyfin-web:latest` | 8096 | `/data`, `/media` (ro) | Jellyfin-compatible media server with the bundled web UI at `/web/`. Image also carries the GPL-2.0 `jellyfin-web` build. Runs as `99:100`. |
| Tinyfin API | `ghcr.io/rafaelvieiras/tinyfin:latest` | 8096 | `/data`, `/media` (ro) | Same server, API only: MIT image with no bundled UI. Use it with the Jellyfin mobile, TV or Kodi clients. Runs as `99:100`. |
| MyMediaDB | `ghcr.io/rafaelvieiras/my-media-db:latest` | 8080 | `/data`, `/media` (rw) | Self-hosted media catalog (scan, match, NFO export, API). `/media` is read-write because the NFO export writes next to the files. No authentication yet. Takes `PUID`/`PGID` (default `99:100`). |
| Playhead TV | `ghcr.io/rafaelvieiras/playheadtv-backend:v0.3.15` | 9394 | `/data` | Linear TV channels assembled on the client, over a Jellyfin library. Backend only (schedule + admin UI). No `latest` tag: the template pins the release. Runs as `99:100`. |

## Status

`M3m0s`, `Tinyfin`, `Tinyfin API`, `MyMediaDB` and `Playhead TV` are the templates so far. Nothing is submitted yet, so the
repository is not registered in CA and the listing does not exist. Locally they can be installed
from the flash drive: `dockerMan/templates-user/` feeds the *Add Container* dropdown, and
`community.applications/private/<folder>/*.xml` feeds the *Private* category in the Apps tab.

## Adding an app

Prerequisites for the project being listed:

- Public repository with an OSI-approved `LICENSE` at the root.
- Image published on a public registry (GHCR or Docker Hub), ideally tagged `:latest`.
- Square **PNG** icon (256×256) committed to the project repository and reachable over a raw
  URL. Keep the `.svg` source beside it, but reference the PNG: Unraid cannot render SVG
  icons (see *Icons must be PNG*).
- No name collision with an existing app at <https://ca.unraid.net/apps>.

Then:

1. Copy `templates/_skeleton.xml.example` to `templates/<app>.xml` and fill every
   placeholder. Keep the `.example` suffix on the skeleton so the CA scanner never
   parses it.
2. Keep `<TemplateURL>` pointing at the real raw URL of that exact file. It is the app
   identity used by the scanner.
3. Use `:latest` as the image tag. Releases then reach users through the image, not
   through template edits. Only edit the template when the configuration surface
   changes: a new env var, port, volume, or description/category update.
4. Validate before committing:

   ```bash
   xmllint --noout ca_profile.xml templates/*.xml
   # Must print nothing: no legacy blocks anywhere.
   grep -nE "<(Networking|Data|Environment)>" templates/*.xml
   ```

   The legacy blocks are not cosmetic. On 17/nov a container recreated from a template
   that had them next to the modern `<Config>` entries came up with an **empty
   environment** and `PortBindings=map[8080/tcp:[{invalid IP 8080}]]`: the app was
   healthy, the container was not. Every port, path and variable belongs in a `<Config>`
   entry and nowhere else. See [`AGENTS.md`](AGENTS.md).
5. Commit and push to `main`.
6. Submit at <https://ca.unraid.net/submit>: sign in, add the repository URL, run
   **Validate**, then **Scan**, preview the listing, and submit. That step is an
   authenticated action on the maintainer account.

## Field notes

- `Name` and `Repository` are the only hard requirements for a Docker entry.
- `Category` is a space-separated list where each token ends in a colon:
  `Productivity: Tools:Utilities`.
- `<WebUI>` refers to the **container** port, not the host port. Unraid rewrites the URL
  when the user remaps the host port.
- Default `PUID`/`PGID` are `99`/`100` (`nobody:users`), otherwise the app cannot write
  to `/mnt/user/appdata`. Prefer exposing them as `<Config>` variables with those
  defaults when the image honours them; fall back to `--user 99:100` in `<ExtraParams>`
  only when it does not.
- Bridge and host networking are the main source of support load. Bake the guidance into
  `<Overview>`/`<Description>`: keep the app bound to `0.0.0.0`, and remember that
  container-name DNS does not work in host mode.
- A template has no repeatable config groups. For apps with N instances, expose one
  instance through env vars in the template and the rest through a mounted config file.

## References

- Submission portal: <https://ca.unraid.net/submit>
- Submission docs: <https://ca.unraid.net/submit/help>
- XML field reference: <https://ca.unraid.net/submit/help/xml-field-reference>
- Starter repository: <https://github.com/unraid/unraid-community-apps-starter>
- Unraid docs: <https://docs.unraid.net/community-applications/>
