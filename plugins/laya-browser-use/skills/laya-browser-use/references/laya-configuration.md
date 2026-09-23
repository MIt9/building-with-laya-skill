# Laya CLI setup

Read this when installing the skill, changing the checkpoint, or diagnosing a `laya-cli` failure. Normal browser tasks call `loadConfig()` and do not choose a checkpoint themselves.

There is no API key, no provider, and no dotenv file — Laya runs locally through `laya-cli`. The only thing to install is the CLI itself, on the `PATH` the CUA/Computer-Use runtime actually executes in (not necessarily your interactive shell):

```bash
uv tool install laya-cli   # or: pipx install laya-cli
laya-cli --version          # -> 0.2.x
```

## Configuration file (optional)

`~/.config/laya-browser-use/config.json` is read by `loadConfig()` and merged into every `decide()` call. It is entirely optional — if missing, `loadConfig()` returns `{}` and `laya-cli` uses its own defaults (English checkpoint, device auto-detected `cuda>mps>cpu`).

```json
{
  "device": "mps",
  "router": true
}
```

- `model` / `subfolder`: pin a specific checkpoint (`convaiinnovations/laya`, `subfolder: "multilingual"`, `subfolder: "typed-decisions"`). Omit for the English default.
- `device`: `cuda` / `mps` / `cpu`. Omit to auto-detect.
- `router`: `true` to auto-route by detected script/language (recommended for non-English sites); when set, `model`/`subfolder` are ignored in favor of `Router(preload=True)`.

Whichever config a session starts with, keep it for that session — switching checkpoints mid-task changes which `laya-cli serve` daemon (if any) answers the request, reintroducing a cold-load pause.

## Always start the daemon first

Every `decide()` call shells out to `laya-cli predict`. Without a resident daemon, the *first* call in a session cold-loads the model (10-35 s on a laptop GPU) — long enough to blow past `decisionTimeoutMs` and surface as `decision_error` instead of a real failure. Before running any browser task:

```bash
laya-cli serve --device mps &      # match the device in config.json, or omit for auto
laya-cli serve status               # wait for {"loaded_at": ...} before starting the Laya loop
```

One daemon per checkpoint config (`hash(model|subfolder|device|router)`); if `config.json` sets `router: true`, start the daemon with `--router` too, or the client hash won't match and every call still cold-loads.

## Diagnose failures by stage

- `laya-cli not found` (`ENOENT` from the subprocess call): the CUA runtime's `PATH` does not include it. Install `laya-cli` in that runtime's own environment, or point `PATH`/use its absolute install location — a working `laya-cli` in your interactive shell does not guarantee the CUA process can see it.
- `laya-cli transport failure or timeout`: the subprocess was killed by `timeoutMs`. Check `laya-cli serve status` first — a missing or dead daemon means the call is cold-loading, not actually hanging. Raise `decisionTimeoutMs` only as a last resort; starting the daemon is the real fix.
- `Invalid laya-cli JSON` / `Invalid laya-cli decision schema`: `laya-cli`'s `--format json` output didn't parse or didn't match the expected Choice contract (`answers.next.choice/confidence/probabilities`). Confirm the installed version with `laya-cli --version` (`0.2.x` expected) and reproduce with `laya-cli predict --state-file <file> --questions-inline '<questions>' --format json` directly to see the raw output.
- Chrome/CUA connector errors are unrelated to Laya: if attachment or login mode fails, that is a [browser-tool discovery](../SKILL.md#discover-the-browser-tool-correctly--required-before-declaring-it-unavailable) problem, not a `laya-cli` one — do not "fix" it by changing checkpoint config.

## References

- Laya model and CLI: the [`laya`](../../../laya/skills/laya/SKILL.md) skill in this marketplace (Choice/Score/Noul primitives, `laya-cli predict`/`serve` contract, daemon lifecycle).
- `laya-cli`: https://github.com/MIt9/laya-cli
