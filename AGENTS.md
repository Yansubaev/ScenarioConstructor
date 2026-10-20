# AGENTS.md

Unity 6 (6000.6), URP, C#. A training-scenario constructor (test assignment). Steps, run modes, logging and analysis are configured in the Unity Editor with no code changes.

- **Requirements**: `docs/task.md`. Read it before scoping or accepting any feature.
- **Architecture plan**: `docs/architecture.md`. Read it before adding or changing a component, and update it when the design changes.

## Rules

- **Commit and push only when the user asks.**
- **Minimal design**: build the fewest classes that meet `docs/task.md`. The grader states that "the number of classes and patterns is not an advantage in itself".
- Use built-in Unity only. Third-party assets, plugins and packages are out; change `Packages/manifest.json` and `ProjectSettings/` only on request.
- Project documents go in `docs/`.

## Unity

- **`.meta` files belong to Unity.** Unity generates them; hand edits break GUIDs. When you move or rename an asset, move its `.meta` with it.
- Make scene, prefab and `.asset` changes in the Editor: give the user steps or write an Editor script. Their YAML references break easily, so hand-edit them only with a clear reason.
- **Input System only.** Legacy `UnityEngine.Input` throws in this project.
- UI is uGUI.
- Code lives in `Assets/Scenario/`, split into Runtime / Editor / Tests asmdefs. Anything that uses `UnityEditor` goes in the Editor assembly.
- Use `[SerializeField] private` fields unless the field needs external access. Renaming a serialized field needs `[FormerlySerializedAs("oldName")]`, or scene data is lost.
- Compiling and running tests happen in the Editor. End each code change by telling the user what to check there, and say plainly that it is unverified.

## Agent skills

- **Issue tracker**: GitHub Issues via `gh`. See `docs/agents/issue-tracker.md`.
- **Triage labels**: the five defaults. See `docs/agents/triage-labels.md`.
- **Domain docs**: single-context (`GLOSSARY.md` + `docs/adr/` at the root). See `docs/agents/domain.md`.
