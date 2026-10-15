# figma-to-android-xml

A [Claude Code](https://claude.com/claude-code) skill that turns Figma designs into production Android XML (Views, not Compose): layouts, drawables, colors, dimens, strings and styles. It follows your project's own conventions and checks the result on a connected device.

It reads designs through the **Framelink Figma MCP server** (`figma-developer-mcp`), so setup is two parts: install the skill, then add Framelink with your Figma token.

> **Quick setup with Claude:** open Claude Code and say
> *"Set up the skill from https://github.com/avadhkachhadiya/figma-to-xml by following its README."*
> Claude follows the steps below and asks you for your Figma token before it adds Framelink.

---

## Requirements

- Claude Code (`claude --version`)
- Node.js 18+ (`node --version`), so `npx` can run Framelink
- Git
- A Figma account with access to the files you want to convert
- Optional: an Android device or emulator on `adb`, for the on-device checks

## 1. Install the skill

Clone the repo straight into your personal skills folder:

```bash
# macOS / Linux / Git Bash
git clone https://github.com/avadhkachhadiya/figma-to-xml.git ~/.claude/skills/figma-to-android-xml
```

```powershell
# Windows PowerShell
git clone https://github.com/avadhkachhadiya/figma-to-xml.git "$env:USERPROFILE\.claude\skills\figma-to-android-xml"
```

If the folder already exists, update it instead: `git -C ~/.claude/skills/figma-to-android-xml pull`.

## 2. Create a Figma personal access token

1. Open **[Figma settings](https://www.figma.com/settings)** (log in first), go to the **Security** tab and scroll to **Personal access tokens**. Figma's guide: [Manage personal access tokens](https://help.figma.com/hc/en-us/articles/8085703771159-Manage-personal-access-tokens).
2. Click **Generate new token** and give it a name, e.g. `claude-framelink`.
3. Scopes: **File content → Read-only** is enough. Also grant **Dev resources → Read-only** if you want.
4. Copy the token right away. Figma only shows it once.

Treat the token like a password: never commit it, paste it into a shared chat, or put it in a project file.

## 3. Add the Framelink MCP server

Replace `YOUR-KEY` with your token. `--scope user` makes Framelink available in every project, which is what the skill expects.

```bash
# macOS / Linux / WSL
claude mcp add Framelink_Figma_MCP --scope user -- npx -y figma-developer-mcp --figma-api-key=YOUR-KEY --stdio
```

```powershell
# Windows (native): npx must run through cmd /c
claude mcp add Framelink_Figma_MCP --scope user -- cmd /c npx -y figma-developer-mcp --figma-api-key=YOUR-KEY --stdio
```

To replace an existing entry (for example, a new token), remove it first: `claude mcp remove Framelink_Figma_MCP --scope user`.

## 4. Verify

```bash
claude mcp list
```

`Framelink_Figma_MCP` should show **✓ Connected**. Restart Claude Code, run `/skills` and check that `figma-to-android-xml` is listed. Then open an Android project and try:

> Implement this screen: https://www.figma.com/design/<fileKey>/<name>?node-id=123-456

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Framelink_Figma_MCP` fails or times out on startup | The first run downloads the package; run `npx -y figma-developer-mcp --version` once, then restart Claude Code. On Windows, make sure you used the `cmd /c` form. |
| `403` / `Invalid token` from Framelink | The token expired, was revoked, or is missing the File content scope. Make a new one and redo step 3. |
| `localPath ... resolves outside the allowed image directory` | Framelink only writes inside the folder Claude Code was started in. The skill uses `build/figma_tmp` in the project and deletes it afterwards. |
| Skill doesn't appear in `/skills` | The folder must be `~/.claude/skills/figma-to-android-xml/` with `SKILL.md` directly inside it. |

---

## Instructions for Claude (when asked to set this up)

Follow steps 1–4 above in order, with these rules:

1. **Before running `claude mcp add`, ask the user for their Figma personal access token** (use the AskUserQuestion tool if available). Explain step 2 and give them the links (https://www.figma.com/settings → Security tab → Personal access tokens) so they know where to create it. Don't continue until they give one; don't invent or reuse a token.
2. Check `claude mcp list` first. If `Framelink_Figma_MCP` already exists, ask whether to keep it or replace it with the new token.
3. Use the command for the user's platform (step 3). On native Windows use the `cmd /c` form.
4. Never echo the token back, write it to any file in a repository, or include it in a commit, log summary or final message. Refer to it as "your token".
5. Install the skill with `git clone` (or `git pull` if the folder exists). Don't edit `SKILL.md` or `reference.md` during setup.
6. Finish by running `claude mcp list`. Report whether Framelink is connected and that the user must restart Claude Code for the skill and server to load.

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | Workflow, rules, device checks and report format |
| `reference.md` | Conversion tables: frame scale, units, typography, opacity, shapes, gradients, shadows, widgets |
| `README.md` | This setup guide (not read by the skill) |
