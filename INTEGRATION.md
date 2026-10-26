# Using Manager UI from your own mod

Manager UI on the Workshop: https://steamcommunity.com/sharedfiles/filedetails/?id=3739704295

Manager UI is open to any Software Inc code mod, not only the Tashus manager mods. Add a page to the Manager
window, a tab to the Manage Team window, lines to the Activity log, and (optionally) an on/off switch, a status
line and a tab strip drawn by the shell. Your mod never references Manager UI's types, so it still loads and runs
when Manager UI is not installed.

## How it works

Mods live in separate assemblies and cannot reference each other's types. Manager UI creates a GameObject named
`ManagerUIHub`; you find it with `GameObject.Find` and call it with `SendMessage(name, object[],
SendMessageOptions.DontRequireReceiver)`. Only `string`, `string[]`, `List<string>` and `System` delegates
(`Action<...>`) are passed, because those are the same types in every mod.

`SendMessage` returns nothing, so reads pass a `List<string>` that the hub fills.

**Compatibility promise:** existing messages and argument positions do not change. New arguments are only ever
appended and are optional, so a mod written against an older Manager UI keeps working, and a mod written against
this one degrades quietly on an older Manager UI (the extra arguments are ignored).

## Messages

| Message | Payload | What it does |
|---|---|---|
| `RegisterPanel` | `{ string name, Action<GameObject> build [, Action<string,string> onRename, Action<bool> setMaster, Action<int> selectTab] }` | A page in the Manager window. `build` draws into the GameObject it is given (a scroll content, top-left origin). Registering again with the same name replaces the entry. |
| `RegisterTeamPanel` | `{ string name, Action<GameObject,string> build }` | A tab in the Manage Team window; `build` gets the team name. |
| `PublishShared` | `{ string key, string[] values }` | Puts values on the shared bus. Publishing an empty array removes the key. Cleared on every scene load. |
| `GetShared` | `{ string key, List<string> outValues }` | Reads a bus key. |
| `ListPanels` | `{ List<string> outNames }` | Registered page names, to detect another mod. |
| `LogActivity` | `{ string modName, string text [, string color "#RRGGBB"] }` | One line in the Activity log; the hub adds the game date. Use your page name as `modName`. |
| `ActivityFeatures` | `{ List<string> outFeatures }` | Contains `"markup1"` when the log understands the markup below. |
| `OpenTeamWindow` | `{ string teamName [, string tabName] }` | Opens the Manage Team window on that team, and on your tab when `tabName` is given. Useful for an "Edit" button on a team row. Older Manager UI versions ignore it. |
| `SaveProfile` | `{ string modKey, string name, string value }` | Stores a named preset. Profiles are kept across saves in Manager UI's settings file. |
| `GetProfile` | `{ string modKey, string name, string[] holder }` | Puts the preset's value in `holder[0]` (pass `new string[1]`). |
| `ListProfiles` | `{ string modKey, List<string> outNames }` | Your preset names. |
| `DeleteProfile` | `{ string modKey, string name }` | Removes a preset. |

`onRename(oldName, newName)` is called when the player renames a team, so a mod that stores team names can follow.

## The shell header (optional)

Without anything below, your page shows its name above whatever you draw. Add any of these to get more:

- **Status.** Publish key `"MUI.Status.<your page name>"` with
  `{ state, doingNow, attention1, attention2, attention3 }` (attention lines optional). `state` is `"on"`,
  `"off"` or `"warn"` (on, but something blocks part of it). The shell shows a dot in the rail, the status
  line and attention lines in your header, and a row on the Overview page.
  **Republish at least every 2 seconds.** A status older than 5 seconds counts as stale ("Not responding"), so
  the shell can tell when a mod was switched off in the Mods window.
- **On/off switch.** Pass `setMaster` (an `Action<bool>`) at index 3 of `RegisterPanel`. The header switch and
  the Overview call it; republish your status after it changes. The switch only appears while your status is
  fresh. Without a setter, draw your own on/off control in your page.
- **Tabs.** Pass `selectTab` (an `Action<int>`) at index 4 and publish `"MUI.Tabs.<your page name>"` as the tab
  names, a name starting with `~` drawn faded (that area is switched off). The shell calls `selectTab` with the
  chosen tab before every `build`, so `build` draws only that tab. The choice is remembered for the session.
- **Team summary.** Publish `"MUI.TeamSummary.<your team tab name>"` as `"team name|one line"` entries; the
  Manage Team window shows your line for that team above the tabs.

Names in rail and Overview drop a trailing " Manager" ("Office Manager" shows as "Office"). The page title
keeps the full name.

## Activity markup

When `ActivityFeatures` lists `"markup1"`, a line can mark parts of its text:

- `\u0001` + kind + text + `\u0002`, where kind is `n` (a name, bold), `i` (money coming in, green) or `o` (money
  going out, orange).
- `\u0003` as the first character marks a skip: something the mod chose not to do, drawn grey and italic.

Without the feature, send plain text. Lines are capped at 300 characters.

## Example

Written for the game's compiler (C# 6: no string interpolation, no pattern matching; avoid properties, see the
note in `ManagerUIHub.cs`).

```csharp
using System;
using UnityEngine;
using UnityEngine.UI;

public class MyModBehaviour : ModBehaviour
{
	private const string PageName = "My Mod";   // keep it stable: it is the bus key and the rail entry
	private bool _on;
	private float _timer;

	private void Update()
	{
		_timer += Time.unscaledDeltaTime;
		if (_timer < 2f) return;
		_timer = 0f;
		GameObject hub = GameObject.Find("ManagerUIHub");
		if (hub == null) return;   // Manager UI not installed: carry on without it
		hub.SendMessage("RegisterPanel", new object[] {
			PageName,
			new Action<GameObject>(Build),
			null,
			new Action<bool>(delegate(bool v) { _on = v; })
		}, SendMessageOptions.DontRequireReceiver);
		hub.SendMessage("PublishShared", new object[] {
			"MUI.Status." + PageName,
			new string[] { _on ? "on" : "off", "Watching 3 teams." }
		}, SendMessageOptions.DontRequireReceiver);
	}

	private void Build(GameObject container)
	{
		Text label = WindowManager.SpawnLabel();
		label.text = "Settings for My Mod go here.";
		WindowManager.AddElementToElement(label.gameObject, container, new Rect(8f, 8f, 400f, 24f), new Rect(0f, 0f, 0f, 0f));
	}

	public override void OnActivate() { }
	public override void OnDeactivate() { }
}
```

## Good to know

- If your `build` throws, the game's own error handling reports it and unloads your mod, as it does for any
  mod that throws. Manager UI shows a short note on the page instead of a blank one, and the rest of the window
  keeps working; your entry then reads "Not responding" once your status stops.
- The bus is current-game only: Manager UI clears it on every scene load, so publish again after a load.
- Pick a page name that is unlikely to clash with another mod; two mods with the same name replace each other.
- Questions, or a message you need added: [open an integration question](https://github.com/Tashus-s-Software-Inc-Mods/Manager-Suite/issues/new?template=integration.yml).
- Once your mod is on the Workshop, [ask for it to be added to the Community Manager Series](https://github.com/Tashus-s-Software-Inc-Mods/Manager-Suite/issues/new?template=collection.yml).
  The same form takes it out again.

## Licence

This document and its example code are under the MIT licence ([LICENSE.md](LICENSE.md)). Copy the example into
your mod freely. Your mod stays yours.
