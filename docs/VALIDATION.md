# Validate UE AngelScript skills

[한국어](VALIDATION.ko-KR.md)

1. Run the development commands in the [README](../README.md). The package test builds the archive twice, compares hashes, checks the three skill names and validates the extracted manifests.
2. Install the extracted ZIP in a fresh Codex environment with no editor MCP connection. Check the installed catalog for all three skills. With the UE MCP companion also installed, check for six distinct skills.
3. In an AngelScript project, resolve a binding against local source, compile a small gameplay change and verify its effect in the editor. Use the project's harness for a world or PIE test. For EmmsUI, use a project with that dependency and check a control in a rendered editor.
4. Record the engine build, installed plugin versions and checks actually performed. Historical compilation, hot-reload and UI observations for the combined v0.1.0 package are in the [original validation record](https://github.com/coquid/unreal-engine-skills-for-codex-plugin/tree/9937d3deeb10e362a7443d849048a9bcd4da7605/docs/SANDBOX_VALIDATION.md).

## Verify in Claude Code

Follow the [installation instructions](../README.md) and open a fresh session. Type `/ue-angelscript-skills:` and check for `angelscript-in-unreal`, `as-integration-tests` and `emmsui`. Use the first skill to locate a local binding and open its Korean reference.

## Split release verification

On 2026-11-17 the standalone v0.1.0 ZIP installed into a fresh Codex client home on Windows. With no MCP endpoint configured, the installed skill catalog returned all three scripting skills. With the UE MCP companion installed, the catalog returned six distinct skills. This run covered installation and discovery. The linked combined-package record documents the earlier gameplay and UI checks.
