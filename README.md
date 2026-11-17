# UE AngelScript Skills for Claude Code and Codex

[한국어](README.ko-KR.md)

Three skills for projects using Unreal Engine with Hazelight AngelScript.

| Skill | Use it for |
| --- | --- |
| `angelscript-in-unreal` | Find local bindings, implement gameplay and verify script changes |
| `as-integration-tests` | Design world, PIE and latent tests using the project's harness |
| `emmsui` | Build developer panels with the EmmsUI plugin installed in the target project |

## Install in Codex

Extract the ZIP from [Releases](https://github.com/coquid/ue-angelscript-skills-plugin/releases) into a new directory named `ue-angelscript-distribution`. From its parent, run:

```sh
codex plugin marketplace add ./ue-angelscript-distribution
codex plugin add ue-angelscript-skills-for-codex@ue-angelscript-skills-for-codex
```

Open a new project chat and ask Codex to use `$angelscript-in-unreal` to locate an existing binding. These skills work directly with local project sources. For live editor operations, optionally install [Unreal Engine MCP Skills](https://github.com/coquid/unreal-engine-skills-for-codex-plugin).

## Install in Claude Code

Register the GitHub marketplace and install the plugin.

```sh
claude plugin marketplace add coquid/ue-angelscript-skills-plugin
claude plugin install ue-angelscript-skills@coquid-as-skills
```

In a new Claude Code session, enter `/ue-angelscript-skills:angelscript-in-unreal`. Both clients load the same `skills/` directory. To install from the release ZIP, replace the repository name in the first command with the relative path to the extracted directory, such as `./ue-angelscript-distribution`.

For GitHub installs, use the marketplace and plugin update actions in Claude Code’s `/plugin` menu.

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
