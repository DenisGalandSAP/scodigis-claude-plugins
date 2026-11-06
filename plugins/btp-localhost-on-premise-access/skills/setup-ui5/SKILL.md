---
name: setup-ui5
description: Set up a SAPUI5 or SAP Fiori elements app (or SAPUI5 library) so it runs and deploys from the local machine through an on-premise SAP BTP destination (Cloud Connector), via a cf ssh tunnel to the connectivity proxy. Use when the user wants to run `npm start` / preview / deploy a UI5 app against an on-premise ABAP system that is only reachable through a BTP destination, add an environment (dev, qual, prod) for that, or fix such a setup (start-btp / deploy-btp scripts, .env.<env>, ui5-btp-<env>.yaml). For ADT / ABAP development use setup-abap; to start an environment that is set up, use run.
argument-hint: "[env]"
---

# Set up a SAPUI5 app

You do what the "BTP On-Premise Access" Template Wizard of SAP Fiori tools does, by talking to the user. Environment
name from the arguments, if any: `$ARGUMENTS`.

Run the CLI `btp-on-premise-setup` from the npm package `@scodigis/generator-btp-on-premise-scripts` as:

```sh
npx -y -p @scodigis/generator-btp-on-premise-scripts@0.8.1 btp-on-premise-setup <command> [options]
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

If the SAP system is reachable directly from the user's machine (no Cloud Connector), they don't need this:
point them to the Service Center of SAP Fiori tools instead. Mention it once at the start.

## 1. Project and environment

1. Find the app: the folder with `package.json`, `ui5.yaml` and `webapp/manifest.json`, or a SAPUI5 library
   (`ui5.yaml` with `type: library`). Usually the current working directory; otherwise ask.
2. `CLI inspect --project <dir>`: the app kind, the environments already set up (`environments`) and, for the
   first of them (or `dev`), the existing values and defaults. On `error`, `candidates` lists app folders found
   underneath; ask which one. If `abap` is true, the folder is set up for ABAP development: use setup-abap.
3. Ask for the environment name (`dev`, `qual`, `prod`...: letters, digits, `-`, `_`). An existing name changes
   that environment, a new one adds another. If it differs from the `env` of the previous call, run
   `CLI inspect --project <dir> --env <name>` again to get its values.
4. If `hasBuildScript` is false, warn that `deploy-btp` needs a `build` script in `package.json`.
5. If `library` is true, it's a SAPUI5 library: nothing to run, it's only deployed. Skip the runtime destination
   (step 4.1 and 4.2) and ask only for the deploy destination; it gets `deploy-btp:<env>`, `deploy-btp-test:<env>`
   and `btp-proxy:<env>`, no `start-btp:<env>`. A hand-written `deploy-btp` script that expects someone else's
   proxy to be running can be replaced by `deploy-btp:<env>`, which starts its own.

## 2. Cloud Foundry login, org, space and tunnel app

Follow `${CLAUDE_PLUGIN_ROOT}/shared/cf-login.md` (sections Login, and Org, space and tunnel app).

## 3. Destinations

List them as in the shared file (section Destinations), then:

1. Ask for the runtime destination (`npm run start-btp:<env>`; not for a library). Default:
   `existing.BTP_DESTINATION`, then one of `runtimeDestinations` from `inspect`, then `deployDefaults.destination`.
2. Ask whether the app should also be deployed through a BTP destination (default yes, unless the environment
   already exists without `BTP_DEPLOY_DESTINATION`). If yes, ask for the deploy destination (default:
   `existing.BTP_DEPLOY_DESTINATION`, then `deployDefaults.destination`, then the runtime destination).

## 4. Deployment (only with a deploy destination)

Defaults come from `deployDefaults` of `inspect`. Ask for, or confirm:

- BSP application name (`--bsp`): up to 15 characters, letters, digits, `_`, optional `/NAMESPACE/`;
- package (`--package`);
- transport request (`--transport`): required unless the package starts with `$` (local, e.g. `$TMP`);
- client (`--client`): three digits; default the destination's `sap-client`, then `deployDefaults.client`.

If the deploy destination has `NoAuthentication`, the upload needs an ABAP user: ask for the user name
(`--deploy-user`). Don't ask for the password in the chat. After step 5 the user enters it in `.env.<env>`
(`SAP_PWD=`) themselves. If the environment already has one (`existing.SAP_PWD` is `***`), it's kept.

## 5. Write

1. Run `apply` with `--dry-run` first and show the user which files it would write or remove:

   ```sh
   CLI apply --project <dir> --env <env> --org <org> --space <space> --app <app> --destination <runtime> \
     [--deploy-destination <deploy> --bsp <name> --package <pkg> [--transport <tr>] [--client <nnn>] [--deploy-user <user>]] \
     [--sso | --no-sso --cf-user <user> --password-env <VAR>] [--origin <origin>] --dry-run
   ```

   For a library, leave out `--destination` and pass `--deploy-destination` with the deployment options.
2. On the user's go-ahead, run the same command without `--dry-run`. It writes the files, adds `.env` and
   `.env.<env>` to `.gitignore`, and runs `cf target` on the org and space.
3. Report the `scripts` it returns and any `warnings`:
   - `npm run start-btp:<env>`: runs the app through the runtime destination (also in VS Code as the run
     configuration "Run with BTP destination (<env>)");
   - `npm run deploy-btp:<env>` / `deploy-btp-test:<env>`: build and deploy, or test mode;
   - `npm run btp-proxy:<env>`: only the proxy.

   `npm start` and the Fiori tools Run / Preview still use `ui5.yaml`. To start the environment from Claude, the
   user can run `/btp-localhost-on-premise-access:run <env>`.
