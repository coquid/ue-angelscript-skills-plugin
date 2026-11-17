# UE AngelScript Skills for Codex

[한국어](README.ko-KR.md)

Three skills for projects using Unreal Engine with Hazelight AngelScript.

| Skill | Use it for |
| --- | --- |
| `angelscript-in-unreal` | Find local bindings, implement gameplay and verify script changes |
| `as-integration-tests` | Design world, PIE and latent tests using the project's harness |
| `emmsui` | Build developer panels with the EmmsUI plugin installed in the target project |

## Install

Extract the ZIP from [Releases](https://github.com/coquid/ue-angelscript-skills-for-codex-plugins/releases) into a new directory named `ue-angelscript-distribution`. From its parent, run:

```sh
codex plugin marketplace add ./ue-angelscript-distribution
codex plugin add ue-angelscript-skills-for-codex@ue-angelscript-skills-for-codex
```

Open a new project chat and ask Codex to use `$angelscript-in-unreal` to locate an existing binding. These skills work directly with local project sources. For live editor operations, optionally install [Unreal Engine MCP Skills](https://github.com/coquid/unreal-engine-skills-for-codex-plugin).

## Migrate from the combined plugin

If `angelscript-unreal-engine-skills-for-codex@angelscript-unreal-engine-skills-for-codex` is installed, remove that plugin and its enablement entry from any project-specific config before enabling the split plugins. Preserve your existing editor MCP connection. Install this package for scripting and the UE MCP package for editor control. The three scripting skill names carry over.

## Development

With Python 3.11+ in a virtual environment, run:

```sh
python -m pip install -r requirements-dev.txt
python scripts/validate_plugin.py
python -m unittest discover -s tests -v
python scripts/package_plugin.py
```

The generated ZIP is in `dist/`. See the [verification procedure](docs/VALIDATION.md), [source and split baseline](UPSTREAM.md) and [MIT license](LICENSE).
