# unraid-templates

Unraid Community Applications (CA) templates for my self-hosted projects.

CA does not host anything. This repository is the metadata layer: a maintainer profile
plus one XML template per container. The Unraid scanner reads the default branch over
raw GitHub URLs and lists the apps in the Apps tab after review.

## Layout

| Path | Purpose |
|------|---------|
| `ca_profile.xml` | Repository profile shown in CA. Must stay at the repository root. |
| `templates/` | One XML file per Docker app. |
| `plugins/` | One XML wrapper per Unraid plugin (none yet). |
| `icon.svg` | Repository icon referenced by `ca_profile.xml`. |

## Status

Skeleton only. No app is submitted yet, so the repository is not registered in CA.

## Adding an app

Prerequisites for the project being listed:

- Public repository with an OSI-approved `LICENSE` at the root.
- Image published on a public registry (GHCR or Docker Hub), ideally tagged `:latest`.
- Square icon committed to the project repository and reachable over a raw URL.
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
4. Run `xmllint --noout ca_profile.xml templates/*.xml` locally.
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
  to `/mnt/user/appdata`.
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
