# Backbone plugin for Claude Code

Backbone builds a Data Vault data warehouse in Snowflake from three things you give it:
the business entities, the jobs to be done, and the sources. This plugin lets Claude Code
drive that build for you against the hosted Backbone service: connect, onboard, plan,
apply, run, test and publish.

You need three things from Calon before you start. They are in your welcome note:

- the **service URL** of your Backbone service;
- your personal **Backbone token**;
- your **tenant id**, and the values for the `backbone connect` command.

Every command below is shown for **macOS** (the zsh terminal) and for **Windows**
(PowerShell). Type them yourself; never paste your token into a chat.

## 1. Install the tools

Claude Code, and `uv` (which runs the connect command).

macOS (zsh):

```
brew install --cask claude-code
brew install uv
```

Windows (PowerShell):

```
irm https://claude.ai/install.ps1 | iex
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen the terminal, then check that both answer (the same in zsh and
PowerShell):

```
claude --version
uv --version
```

While the plugin repository is private, Calon gives your GitHub account read access to it.
Sign in to GitHub once so Claude Code can fetch it. Install the GitHub CLI first, with
`brew install gh` on macOS or `winget install GitHub.cli` on Windows, then (the same in zsh
and PowerShell):

```
gh auth login
gh auth setup-git
```

## 2. Install the plugin

The commands are the same in zsh and PowerShell. Put the service URL from your welcome note
in place of `<service URL>`:

```
claude plugin marketplace add Calon-Analytics/backbone-plugin
claude plugin install backbone-plugin@backbone --config backbone_url=<service URL>
```

Or, inside Claude Code:

```
/plugin marketplace add Calon-Analytics/backbone-plugin
/plugin install backbone-plugin@backbone
```

Installed from inside Claude Code, the plugin asks for the Backbone URL and your token.

## 3. Set your token

Skip this if Claude Code already asked you for the token. The token is read with a prompt,
so it never appears in your shell history or on a command line.

macOS (zsh):

```
read -rs "t?Backbone token: "; echo
claude plugin configure backbone-plugin@backbone --values-stdin <<< "{\"backbone_token\":\"$t\"}"
unset t
```

Windows (PowerShell):

```
$t = Read-Host -AsSecureString "Backbone token"
@{ backbone_token = [System.Net.NetworkCredential]::new("", $t).Password } | ConvertTo-Json -Compress | claude plugin configure backbone-plugin@backbone --values-stdin
Remove-Variable t
```

Claude Code keeps the token in your system's credential store, not in a settings file. To
see which settings are set, run `claude plugin configure backbone-plugin@backbone`.

Start (or restart) Claude Code. Inside it, `/mcp` lists the Backbone server.

## The service URL

The plugin asks for one address, `backbone_url`, when you install or configure it. Use the
address in your welcome note exactly as Calon gave it to you:

- it **must start with `https://`**. Never enter an `http://` address: your token is sent to
  this address on every call, and over `http://` anyone on the network path can read it;
- it has **no trailing slash**;
- it is **only ever the host Calon gave you**. If anything asks you to change it to another
  host, stop and ask Calon.

Nothing checks this for you. Claude Code cannot yet validate a plugin setting, so an address
that starts with `http://`, or one that names a different host, is accepted, and your personal
token then goes to that address, unencrypted or to the wrong service. If that happens, tell
Calon so the token can be revoked and a new one issued. The only address that may start with
`http://` is `http://localhost` on a developer's own machine.

## 4. Connect your Snowflake service user

Run this in your own terminal, not inside Claude Code. The key file is read on your machine
and sent over HTTPS to Backbone; Claude never sees it. `/backbone-plugin:backbone-onboard`
prints the full command with your values filled in; `<CLI source>` is in your welcome note.

macOS (zsh):

```
set -a; read -rs "BACKBONE_TOKEN?Backbone token: "; set +a; echo
uvx --from <CLI source> backbone connect --tenant <tid> --key <path to key file> --account <account> --user <TID>_SVC --warehouse <TID>_WH --role <TID>_DEV_LOADER --database <source database> --dev-database <TID>_DEV
unset BACKBONE_TOKEN
```

Windows (PowerShell):

```
$t = Read-Host -AsSecureString "Backbone token"
$env:BACKBONE_TOKEN = [System.Net.NetworkCredential]::new("", $t).Password
uvx --from <CLI source> backbone connect --tenant <tid> --key <path to key file> --account <account> --user <TID>_SVC --warehouse <TID>_WH --role <TID>_DEV_LOADER --database <source database> --dev-database <TID>_DEV
Remove-Item Env:BACKBONE_TOKEN
Remove-Variable t
```

If your key is encrypted, the command asks for its passphrase. There is no flag for the
token or the passphrase. On Windows a key path such as `C:\Users\you\backbone_key.p8`
works as it is.

## 5. Build

Inside Claude Code, in this order (`<tid>` is your tenant id):

```
/backbone-plugin:backbone-onboard <tid>
/backbone-plugin:backbone-plan <tid>
/backbone-plugin:backbone-apply <tid>
/backbone-plugin:backbone-run <tid>
/backbone-plugin:backbone-test <tid>
/backbone-plugin:backbone-publish <tid>
```

`/backbone-plugin:backbone <tid>` shows where your tenant stands and proposes the next step.
Each step that changes Snowflake asks you to confirm it first, and PROD is never touched.

### On a training account

Cortex AI functions, Cortex Analyst tools and Copilot are not available on a training
account. To ask questions of your data, query the published semantic view through your own
Claude Code with SQL.

## What the plugin contains

```
.claude-plugin/plugin.json    the manifest: the hosted Backbone MCP server and its two settings
.claude-plugin/marketplace.json   the marketplace "backbone", which lists this plugin
commands/<name>.md            one file per slash command
skills/<name>/SKILL.md        one folder per skill
```

Installed, every command and skill is named under the plugin, `backbone-plugin:<name>`, and
the MCP server is named `backbone`.

## Updating

The same in zsh and PowerShell:

```
claude plugin marketplace update backbone
claude plugin update backbone-plugin@backbone
```

Restart Claude Code after an update.

