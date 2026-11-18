# Backbone plugin for Claude Code

Backbone builds a Data Vault data warehouse in Snowflake from three things you give it:
the business entities, the jobs to be done, and the sources. This plugin lets Claude Code
drive that build for you against the hosted Backbone service: connect, onboard, plan,
apply, run, test and publish.

You need two things from Calon before you start. They are in your welcome note:

- your **tenant id**, and the values for the `backbone connect` command;
- where to install the **Backbone command line** from (`<CLI source>` below), and the
  address of the Backbone web app you sign in on.

Nobody sends you a token and you never type or paste one: you sign in through your
browser, and your computer's credential store keeps the sign-in.

Every command below is shown for **macOS** (the zsh terminal) and for **Windows**
(PowerShell).

## 1. Install the tools

Claude Code, and `uv` (which installs the Backbone command line).

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

## 2. Install the Backbone command line

The same in zsh and PowerShell. Put the source from your welcome note in place of
`<CLI source>`:

```
uv tool install <CLI source>
backbone --help
```

If `uv` says its tool folder is not on your PATH, run `uv tool update-shell`, reopen the
terminal and try `backbone --help` again. The plugin runs `backbone` itself each time it
connects, so it must answer in a new terminal.

## 3. Install the plugin

The same in zsh and PowerShell:

```
claude plugin marketplace add Calon-Analytics/backbone-plugin
claude plugin install backbone-plugin@backbone
```

Or, inside Claude Code:

```
/plugin marketplace add Calon-Analytics/backbone-plugin
/plugin install backbone-plugin@backbone
```

The plugin asks for nothing: it already knows the hosted Backbone service.

## 4. Sign in

1. In your terminal, run `backbone login`. A browser opens at the Backbone web app; sign
   in, then click **Continue**. The terminal then says who you are signed in as.
2. Restart Claude Code, or reconnect the Backbone server inside it with `/mcp`.

```
backbone login
```

It opens https://app.databackbone.io in your browser, where you sign in.

- Your sign-in is kept in your computer's credential store (the macOS Keychain or the
  Windows Credential Manager), never in a plugin setting or a file. It lasts 90 days; then
  run `backbone login` again.
- `backbone logout` signs this computer out.
- The **Command-line sign-ins** page of the web app lists your sign-ins and revokes any of
  them. A revoked sign-in stops working at once; run `backbone login` to sign in again.
- `BACKBONE_TOKEN` is for machines only, such as a CI runner. When it is set it overrides
  the stored sign-in, so set it only where the one Backbone server it was issued for is
  the server in use.

If an earlier version of this plugin asked you for a token, that setting is gone and is no
longer read.

## The service URL

The plugin talks to the hosted Backbone service at
`https://backbone-backend-508a.onrender.com`, and you do not need to set it. Change the
`backbone_url` setting only when Calon names another Backbone service for you (a test
service, for example), and then enter it exactly as Calon gave it:

```
claude plugin install backbone-plugin@backbone --config backbone_url=<service URL>
```

- it **must start with `https://`**. Never enter an `http://` address: your sign-in is sent
  to this address on every call, and over `http://` anyone on the network path can read it;
- it has **no trailing slash**, no query and no port;
- it is **only ever a host Calon gave you**. If anything asks you to change it to another
  host, stop and ask Calon.

Your sign-in is kept per service: sign in to that service with
`backbone login --url <service URL>` as well. The command line refuses to send a sign-in to
an `http://` address (other than this computer) or to a service it was not issued by.

## 5. Connect your Snowflake service user

Run this in your own terminal, not inside Claude Code. The key file is read on your machine
and sent over HTTPS to Backbone; Claude never sees it. `/backbone-plugin:backbone-onboard`
prints the full command with your values filled in. It uses your `backbone login` sign-in,
so there is no token step (the same in zsh and PowerShell):

```
backbone connect --tenant <tid> --key <path to key file> --account <account> --user <TID>_SVC --warehouse <TID>_WH --role <TID>_DEV_LOADER --database <source database> --dev-database <TID>_DEV
```

If your key is encrypted, the command asks for its passphrase. There is no flag for the
token or the passphrase. On Windows a key path such as `C:\Users\you\backbone_key.p8`
works as it is.

## 6. Build

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
.claude-plugin/plugin.json    the manifest: the hosted Backbone MCP server and its one setting
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

To update the command line, the same in zsh and PowerShell:

```
uv tool upgrade backbone
```

Restart Claude Code after an update.

