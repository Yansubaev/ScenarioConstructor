# AGENTS.md

Unity 6 (6000.6) project, URP, C#. A training-scenario constructor (test assignment): steps, run modes, logging and analysis are configured in the Unity Editor without code changes.

- Task: `docs/task.md`
- Architecture: `docs/architecture.md`

## Rules

- **Do not commit or push** unless the user explicitly asks for it.
- Keep it simple. Evaluation criterion: "the number of classes and patterns is not an advantage in itself". No speculative abstractions.
- No third-party assets, plugins or packages; only built-in Unity features. Do not edit `Packages/manifest.json` without being asked.
- Project documents go in `docs/` (except `README.md` and `AGENTS.md`). `docs/` is outside `Assets/`, so Unity does not create `.meta` files for it.

## Unity specifics

- **Do not create, delete or edit `.meta` files by hand.** Unity generates them. When moving or renaming an asset, move its `.meta` too, otherwise GUIDs and references break.
- Do not touch `Library/`, `Temp/`, `Logs/`, `UserSettings/`: they are generated and gitignored.
- Do not edit `.unity`, `.prefab` or `.asset` (YAML) by hand without a good reason. It is easy to break references. Prefer describing the changes to make in the Editor, or use an Editor script.
- Do not change `ProjectSettings/` without being asked.
- Input: **Input System only** (`activeInputHandler: 1`). The legacy `UnityEngine.Input` does not work.
- UI: uGUI (`com.unity.ugui`).
- Code goes in `Assets/Scenario/` with asmdefs (Runtime / Editor / Tests). Editor code goes only in the Editor assembly.
- Serialization: `[SerializeField] private` fields instead of public ones where external access is not needed. Do not rename serialized fields without `[FormerlySerializedAs]`, or data in scenes is lost.
- Code cannot be compiled or tests run outside the Editor without Unity; say so honestly and ask the user to check in the Editor.
