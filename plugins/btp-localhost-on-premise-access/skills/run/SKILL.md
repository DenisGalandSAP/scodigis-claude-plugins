---
name: run
description: Start a SAPUI5 or SAP Fiori elements app, or only the proxy (also the proxy of an ABAP system for ADT), through an on-premise SAP BTP destination already set up with setup-ui5 or setup-abap, logging in to Cloud Foundry first when the session has expired. Also deploys a UI5 app through the destination when the user asks for it. Use for "run / start / preview the app through BTP", "start the proxy", "start ADT access to <system>", "deploy through BTP", npm run start-btp:<env>, btp-proxy:<env>, deploy-btp:<env>.
argument-hint: "[env] [start | proxy | deploy | deploy-test]"
---

# Run through BTP

Starts an environment set up by setup-ui5 or setup-abap. The npm scripts in the project do the real work (proxy, `cf ssh`
tunnel, `fiori run` or `fiori deploy`); you check the login first and run them in the background.

`CLI` stands for:

```sh
npx -y -p @scodigis/generator-btp-on-premise-scripts@0.8.0 btp-on-premise-setup <command> [options]
```

Arguments: `$ARGUMENTS` (environment name and/or mode, both optional).

## 1. Project and environment

1. The project is the current working directory, unless the user named another folder.
2. `CLI inspect --project <dir>`. If `environments` is empty, nothing is set up yet: offer
   `/btp-localhost-on-premise-access:setup-ui5` (an app) or `:setup-abap` (ABAP development) and stop.
3. Environment: the one named in the arguments, else the only one, else ask (AskUserQuestion, `dev` first).
4. Mode: `start` (default) runs the app, `proxy` only the proxy, `deploy` / `deploy-test` build and deploy. A
   SAPUI5 library (`library: true`) has no `start`: offer `proxy` or `deploy`. An ABAP folder (`abap: true`) only
   has `proxy`: use it without asking, and once it listens, give the ADT URL `https://127.0.0.1:<tls port>`
   (`[btp-proxy] listening on https://...`).

## 2. Cloud Foundry login

1. `CLI status --project <dir> --env <env>`. Next to the current session (`loggedIn`, `api`, `user`), it shows
   the environment (`environment.api`, `.org`, `.space`, `.sso`), the user (`envUser`) and password variable
   (`passwordEnv`) the scripts log in with, and whether that variable is set (`passwordSet`). It never shows the
   password.
2. Logged in (`loggedIn` true) to the environment's endpoint (`api` equal to `environment.api`, or the
   environment has no `api`, from an older setup): go on. The scripts switch org / space themselves.
3. Not logged in, or another endpoint:
   - `envCredentials` true: go on, the scripts run `cf auth` themselves.
   - `environment.sso` true: the scripts cannot log in without a terminal. Ask the user to run
     `cf login -a <api> -o <org> -s <space> --sso` in their own terminal (or in this session with the `!` prefix),
     then check `CLI status` again.
   - otherwise: `envUser` is empty (no user for this environment and no `CF_USERNAME`): run setup-ui5 / setup-abap
     again for this environment. `passwordSet` false: give the user the snippet for `passwordEnv` from
     `${CLAUDE_PLUGIN_ROOT}/shared/cf-login.md` (section Login), to run in their own terminal, outside Claude, then restart Claude. Never ask for the
     password in the chat. Stop here.

## 3. Start

Run the npm script from the project folder with the Bash tool in the background (`run_in_background: true`), so
the session stays free while the app runs:

| Mode | Command |
|---|---|
| `start` | `npm run start-btp:<env>` |
| `proxy` | `npm run btp-proxy:<env>` |
| `deploy` | `npm run deploy-btp:<env>` |
| `deploy-test` | `npm run deploy-btp-test:<env>` |

`deploy` writes to the ABAP system (BSP application, package and transport in `ui5-deploy-btp-<env>.yaml`): run it
only when the user asked for a deployment, and confirm the target first. Prefer `deploy-test` when they only want
to check it.

Then watch the output until one of these shows up:

- `[btp-proxy] listening on http://localhost:<port>`: the proxy is up (enough for `proxy`).
- the `fiori run` server URL (usually `http://localhost:8080`): the app is up. Give the user the URL with the page
  it opens.
- a deployment result, for `deploy` / `deploy-test`.
- an error line starting with `[btp]` or `[btp-proxy]`: relay it as is. Common ones:
  - `cf auth as <user> failed` or `was rejected before with this password`: wrong or expired password in the
    variable. It is not tried again until the variable changes. The user updates it (snippet again)
    and restarts Claude.
  - `logged in as <user>, but cf target ... failed`: the user lacks a role in that org / space, or it was
    renamed. Run setup-ui5 / setup-abap again for this environment.
  - `App '<app>' must be bound to a destination and a connectivity service`: see
    `${CLAUDE_PLUGIN_ROOT}/shared/cf-login.md` (section Org, space and tunnel app).
  - `port <n> is taken`: another proxy or app is running. The scripts replace a stale btp-proxy themselves.
    Anything else on the port the user has to stop.

## 4. Stop

When the user is done, stop the background task. The scripts stop the proxy they started with it. A proxy
started with `proxy` mode runs until its task is stopped.
