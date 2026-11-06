---
name: setup-abap
description: Set up a folder for ABAP development on an on-premise SAP system that is only reachable through an SAP BTP destination (Cloud Connector exposing /sap/bc/adt): a local proxy with an https listener for ABAP Development Tools in VS Code or Eclipse, the localhost certificate they trust, and optionally an ADT MCP server (vsp) for Claude. Use when the user wants ADT, Eclipse ADT or an ABAP MCP server against such a system, add a system or client for that, or fix such a setup. For SAPUI5 apps use setup-ui5; to start the proxy of a system that is set up, use run.
argument-hint: "[env, e.g. d01-100]"
---

# Set up ABAP development

The proxy forwards ADT's requests through the cf ssh tunnel and the Cloud Connector, with the destination's
credentials. Environment name from the arguments, if any: `$ARGUMENTS`.

Run the CLI `btp-on-premise-setup` from the npm package `@scodigis/generator-btp-on-premise-scripts` as:

```sh
npx -y -p @scodigis/generator-btp-on-premise-scripts@0.8.3 btp-on-premise-setup <command> [options]
```

Below, `CLI` stands for that command. Every command prints JSON; a failure prints `{ "error": "..." }` and exits
non-zero. Quote option values that contain spaces. Requirements: Node.js 22+ and the cf CLI in `PATH`. The CLI does
all the reading and writing: never write the generated files (`scripts/btp*.js`, `.env.*`...) by hand.

Use AskUserQuestion whenever the user has to pick from a list (region, org, space, app, destination) and offer
the values the CLI returned, with the default first. Skip a question when there is only one valid answer, and
say what you picked.

## Plugin configuration

The user's defaults, from the plugin configuration (`/plugin` > Installed > btp-localhost-on-premise-access >
Configure). A value that is empty, or still reads `${user_config...}`, is not configured:

| Setting | Value |
|---|---|
| API endpoint | `${user_config.cf_api}` |
| Org | `${user_config.cf_org}` |
| Space | `${user_config.cf_space}` |
| Tunnel app | `${user_config.cf_app}` |
| User | `${user_config.cf_user}` |
| Password variable | `${user_config.cf_password_env}` |
| Identity provider origin | `${user_config.cf_origin}` |
| Single sign-on | `${user_config.cf_sso}` |

Use them as defaults after the environment's own values (`existing.*`), and say which ones came from the
configuration. Never ask for or show a password: the configuration only holds the NAME of the variable that has
it. If nothing is configured and the user sets up environments often, mention once that filling in the
configuration saves the questions.

The Cloud Foundry steps are the same for SAPUI5 and ABAP. Read `${CLAUDE_PLUGIN_ROOT}/shared/cf-login.md` before
step 2 and follow it there.

If the SAP system is reachable directly from the user's machine, they don't need this: ADT connects to it as is.
Mention it once at the start.

## 1. Folder and environment

1. The folder: any folder without `ui5.yaml`, usually one per customer or system landscape (not a UI5 project).
   It can be empty: `package.json` is created. Default: the current working directory if it has no `ui5.yaml`,
   otherwise ask.
2. `CLI inspect --project <dir> --abap`: the environments already set up and their values. A folder set up
   before is recognised without the flag (`abap: true`). On `error` about a SAPUI5 project, ask for another folder.
3. Ask for the environment name: one per system and client, e.g. `d01-100`, `d01-300` (letters, digits, `-`,
   `_`). Each gets its own ports. If it differs from the `env` of the previous call, run
   `CLI inspect --project <dir> --abap --env <name>` again.

## 2. Cloud Foundry login, org, space and tunnel app

Follow `${CLAUDE_PLUGIN_ROOT}/shared/cf-login.md` (sections Login, and Org, space and tunnel app).

## 3. Destination

List the destinations as in the shared file (section Destinations), and ask for the ABAP system's one. Default:
`existing.BTP_DESTINATION`. It has to expose `/sap/bc/adt` in the Cloud Connector, which a setup made for Fiori
apps often doesn't: if ADT later answers 403 on `/sap/bc/adt/...`, that's where to look. There's no deployment.

## 4. Write

1. `apply` with `--dry-run` first, and show the user which files it would write:

   ```sh
   CLI apply --project <dir> --env <env> --abap --org <org> --space <space> --app <app> --destination <dest> \
     [--sso | --no-sso --cf-user <user> --password-env <VAR>] [--origin <origin>] --dry-run
   ```
2. On the user's go-ahead, run it without `--dry-run`. It writes `scripts/btp-env.js`, `scripts/btp-proxy.js`,
   `scripts/btp-mcp.js`, `.env.<env>` (with `BTP_PROXY_TLS_PORT`: the http port + 393, e.g. 5050 / 5443) and
   `package.json` with `btp-proxy:<env>`, and runs `cf target` on the org and space.

## 5. Certificate for ADT

ADT only accepts https, so the proxy's https listener needs a localhost certificate that ADT's Java runtime trusts.

1. `CLI adt-cert`: creates it once (`~/.config/btp-proxy`) and imports it into the JRE of ABAP Development Tools
   for VS Code. Report each `vscode` entry: after an import, the user reloads the VS Code window.
2. For each `eclipse` entry, give the user its `command`, to run in PowerShell (`shell: powershell`) or a terminal
   with Eclipse closed. `installs` lists the Eclipse folders that run on that Java (several can share one through
   `-vm` in `eclipse.ini`), so one command covers them all. If the user's Eclipse isn't listed (a folder name that
   doesn't start with `eclipse`), ask where it is and which Java its `eclipse.ini` names with `-vm`.
3. Then they add the system as `https://127.0.0.1:<BTP_PROXY_TLS_PORT>` (VS Code ADT: a destination with the
   client; Eclipse: an ABAP Cloud Project), once the proxy runs (`npm run btp-proxy:<env>`, or
   `/btp-localhost-on-premise-access:run <env> proxy`).
4. After an ADT extension or Eclipse update (a new JRE folder), run `adt-cert` again.

## 6. ADT MCP server (optional)

Offer an MCP server so Claude can read (or change) ABAP objects through the same proxy. It uses vsp; ask for:

- the vsp binary (default `vsp` in `PATH`);
- the ABAP user: the destination's `User` (the proxy logs on with the destination's credentials, the user name
  only labels the connection);
- the client: the destination's `sap-client`;
- read-only, or the packages it may change (`--allowed-packages`, e.g. `Z*,$TMP`);
- the scope: `local` (default, this folder only) or `user` (every folder).

Run `CLI adt-mcp --project <dir> --env <env> --vsp <path> --user <user> --client <nnn> [--read-only]
[--allowed-packages <list>] [--scope <scope>]`, show the `command` it prints, and on the user's explicit yes run
it as is: it registers the server with Claude Code. The launcher starts the proxy in the background when none
runs, so the MCP needs no terminal. A cf login it may need comes from the environment's password variable, so
Claude has to be started from a shell that has it. The new server shows up after a restart of Claude Code.
