# BTP Localhost On-Premise Access

Reach an on-premise SAP system from your own machine through an SAP BTP destination and the Cloud Connector. A
local proxy goes through a `cf ssh` tunnel to the connectivity proxy of a Cloud Foundry app, adds the
destination's credentials and forwards each request. It works for SAPUI5 and SAP Fiori elements apps (run and
deploy) and for ABAP development (ABAP Development Tools in VS Code or Eclipse, and ADT MCP servers). It runs on
Windows, macOS and Linux.

```text
/plugin marketplace add DenisGalandSAP/scodigis-claude-plugins
/plugin install btp-localhost-on-premise-access@scodigis
```

## Commands

| Command | Use |
|---|---|
| `/btp-localhost-on-premise-access:setup-ui5 [env]` | Set up an environment (`dev`, `qual`...) of a SAPUI5 / SAP Fiori elements app or library |
| `/btp-localhost-on-premise-access:setup-abap [env]` | Set up a folder for ABAP development on a system and client (`d01-100`...) |
| `/btp-localhost-on-premise-access:run [env] [start \| proxy \| deploy \| deploy-test]` | Start the app, the proxy or a deployment in the background |

You can also just ask, for example "set up this app to run through our BTP destination for dev" or "give me ADT
access to D01 client 100".

## Configuration

In `/plugin`, select the plugin in the Installed tab, then Configure. Every field is an optional default for new
environments: API endpoint, org, space, tunnel app, user (e-mail), password variable, identity provider origin
and single sign-on.

The password variable holds a name, such as `CF_PASSWORD` or `ACME_CF_PASSWORD`, never the password. You set the
variable itself in your shell profile; the setup commands give you a snippet that reads it without showing it.
Each environment keeps its user and variable name in `.env.<env>` (`BTP_CF_USER`, `BTP_CF_PASSWORD_ENV`), so
projects for different customers can log in as different users.

## SAPUI5 apps

Claude asks for the environment, the Cloud Foundry region, org, space and app, and the runtime and deploy
destinations. It shows a dry run, then writes into the project:

- `scripts/btp-proxy.js`, `scripts/btp.js` and `scripts/btp-env.js`, the proxy and the start and deploy wrappers
- `ui5-btp-<env>.yaml` and `ui5-deploy-btp-<env>.yaml`
- `.env.<env>` with the settings, added to `.gitignore`
- the npm scripts `start-btp:<env>`, `deploy-btp:<env>`, `deploy-btp-test:<env>` and `btp-proxy:<env>`

## ABAP development

The system's destination has to expose `/sap/bc/adt` in the Cloud Connector. Use a folder of its own, one per
customer or landscape, with one environment per system and client. Each environment gets:

- `npm run btp-proxy:<env>`, the proxy on `http://127.0.0.1:<port>` and `https://127.0.0.1:<port + 393>`. ADT only
  accepts https. The proxy answers ADT's `/sap/public/bc/icf/virtualhost` probe itself, because the Cloud
  Connector usually blocks `/sap/public`.
- `scripts/btp-mcp.js`, which starts the proxy in the background when none runs and then the MCP server, since
  Claude Code starts MCP servers without a terminal.

Claude then creates a localhost certificate and imports it into the Java runtime of ABAP Development Tools for VS
Code. For Eclipse it gives you the `keytool` command to run with Eclipse closed. You add the system in ADT as
`https://127.0.0.1:<port + 393>` (an ABAP Cloud Project in Eclipse). If you want an ADT MCP server, Claude shows
you the `claude mcp add` command for [vsp](https://github.com/oisee/vibing-steampunk) behind the proxy and runs it
when you agree.

## Cloud Foundry login

Claude logs in with the user and password variable of the environment, or with a single sign-on passcode when you
ask for it. When the session expires, the scripts run `cf auth` on their own, also in a proxy started in the
background, and the proxy logs in again before it reopens a dropped tunnel. A rejected password is remembered (as a
short fingerprint, never the password) and not tried again until the variable changes, so a wrong or rotated
password doesn't end up locking your account. Single sign-on sessions can't be renewed without you: run
`cf login --sso` in a terminal when they expire.

## Requirements

- Node.js 22 or later and the [cf CLI](https://github.com/cloudfoundry/cli) v8
- A BTP subaccount with the Destination and Connectivity services, a connected Cloud Connector, and OnPremise
  destinations with `NoAuthentication` or `BasicAuthentication`
- The SpaceDeveloper role in the Cloud Foundry space, with SSH allowed. If no app in the space is bound to both
  services, Claude can create a small `btp-tunnel` app for it (with your agreement).
- For ADT: OpenSSL (on Windows it comes with Git for Windows)

The plugin runs the command-line tool of the npm package
[`@scodigis/generator-btp-on-premise-scripts`](https://github.com/DenisGalandSAP/generator-btp-on-premise-scripts)
through `npx`. It is the same code as the "BTP On-Premise Access" generator of the SAP Fiori tools Template Wizard.

## Data handling

Everything runs on your machine, and nothing is sent to the plugin's author. The tool and the scripts it writes
connect to:

- the Cloud Foundry API and login server of your region, through the cf CLI
- the Destination service and its token endpoint in your subaccount, with the credentials of the app's service
  binding
- the connectivity proxy of your subaccount, through the `cf ssh` tunnel, which forwards requests to the Cloud
  Connector and your backend
- `raw.githubusercontent.com`, for SAP's public list of Cloud Foundry regions (a plain download)
- the npm registry, when `npx` downloads the package

`.env.<env>` holds the org, space, app and destination names, your Cloud Foundry user and the name of its password
variable, and, only when you type it there yourself, the ABAP password for a deploy destination without
authentication. Destination passwords and client secrets are never shown to Claude. The Cloud Foundry password
stays in your shell environment; Claude only sees whether the variable is set.

## License

[MIT](LICENSE)
