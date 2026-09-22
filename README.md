# Content Pipeline Tools documentation

The published documentation for **Content Pipeline Tools**, an Unreal Engine 5.8 editor plugin: bake
a material subgraph down to a texture, batch repeated props into instanced components inside one
culling bound, merge collision bodies, override the bounds an actor is culled by, and three texture
and navmesh utility windows.

The site is `index.md`, which is the buyer-facing README shipped inside the plugin. Keeping one
source is deliberate: a plugin whose documentation disagrees with itself is worse than one with none,
and this way the page cannot drift from what the buyer already has in
`Plugins/ContentPipelineTools/README.md`.

## Publishing this

1. Create a public repository named `ContentPipelineTools-docs`.
2. Push these five files.
3. **Settings → Pages → Build and deployment → Deploy from a branch → `main` / `(root)`**.
4. The address is `https://<user>.github.io/ContentPipelineTools-docs/`. It is already written into
   the plugin's `DocsURL`; open it once and confirm it answers before the listing goes in.

Nothing needs installing. The theme is `remote_theme`, which GitHub builds on its side; the one file
of it kept here is the layout, so that the page carries no link back into this repository.

## Updating it

`index.md` is a copy of the plugin README plus two things: the YAML front matter and the `## On this
page` list, both in the first thirty lines. `Scripts/MakeDocsSite.ps1` in the plugin repository
regenerates it from the README so the copy cannot go stale by hand.

| file | what it is |
|---|---|
| `index.md` | the documentation page — the plugin README plus front matter and a contents list |
| `third-party-licenses.md` | the notices that ship with the plugin |
| `_config.yml` | title, description, and the GitHub-hosted theme |
| `_layouts/default.html` | the theme's own layout with the “View on GitHub” button taken out |
| `icon.png` | the plugin icon, 128×128 |
