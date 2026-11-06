# Cloud Foundry login, org, space and tunnel app

Shared by the setup-ui5 and setup-abap skills. `CLI` is the command named in the skill, "the configuration" is
the plugin configuration table in the skill, and `existing.*` are the environment's values from `CLI inspect`.

## Login

1. `CLI status`.
2. If `loggedIn` is true and the environment has no `existing.BTP_CF_API` or the same one as `api`, ask whether
   to use the current session (show `api` and `user`) or log in to another region / subaccount.
3. To log in, find the API endpoint: `existing.BTP_CF_API`, else the configured API endpoint, else the current
   `api`. To let the user choose, `CLI regions` lists the regions (`key`, `location`, `provider`, `api`). The user
   can also give an endpoint from the subaccount overview in the BTP cockpit. Then:
   - **User and password from an environment variable** (default, unless single sign-on is `true` in the
     configuration):
     - the user is `existing.BTP_CF_USER`, else the configured user, else ask;
     - the variable is `existing.BTP_CF_PASSWORD_ENV`, else the configured password variable, else `CF_PASSWORD`;
     - the origin is `existing.BTP_CF_ORIGIN`, else the configured origin, else none (SAP ID service).

     Run `CLI status --user <user> --password-env <VAR>`. If `passwordSet` is true, run
     `CLI login --api <api> --user <user> --password-env <VAR> [--origin <origin>]`. A rejected password is
     remembered (for that password only) and not tried again, so the account does not get locked: the user has to
     update the variable.

     If `passwordSet` is false, never ask for the password in the chat. Give the user a snippet to run in their
     own terminal, outside Claude, that reads it without echo, so it never lands on a command line or in the shell
     history (replace VAR):
     - Linux, macOS, WSL (a file only they can read, loaded by `~/.bashrc`, or `~/.zshrc` for zsh):
       ```bash
       case "$SHELL" in *zsh) rc=~/.zshrc ;; *) rc=~/.bashrc ;; esac; mkdir -p ~/.config && (umask 077; read -rsp 'Password for VAR: ' v && echo && printf 'export VAR=%q\n' "$v" >> ~/.config/secrets.env) && chmod 600 ~/.config/secrets.env && { grep -qs 'config/secrets.env' "$rc" || echo '[ -f ~/.config/secrets.env ] && . ~/.config/secrets.env' >> "$rc"; }
       ```
     - Windows (PowerShell, user environment variable):
       ```powershell
       $p = Read-Host 'Password for VAR' -AsSecureString; [Environment]::SetEnvironmentVariable('VAR', [Net.NetworkCredential]::new('', $p).Password, 'User'); Remove-Variable p
       ```

     Claude Code only sees the variable once restarted from a new terminal. Until then the user can run
     `cf login -a <api>` in their terminal instead.

     For `apply`, pass `--no-sso --cf-user <user> --password-env <VAR>`, plus `--origin` when one was used
     (`--origin ""` removes an origin kept from an earlier setup). With these, the generated scripts log in again
     on their own (`cf auth`) when they start with an expired session, and the proxy does the same when it reopens
     a dropped tunnel, also in the background where nothing can prompt.
   - **Single sign-on**, only when the user asks for it or the configuration says so: `CLI login-url --api <api>`
     and give the user `passcodeUrl`. When they paste the one-time passcode:
     `CLI login --api <api> --passcode <code>`. Pass `--sso` to `apply`. The scripts can't log in on their own then:
     an expired session needs `cf login --sso` in a terminal.
   - If the session was kept as it is, pass neither flag: the environment keeps what it had.

## Org, space and tunnel app

1. `CLI orgs`, then `CLI spaces --org <org>`. Defaults: `existing.BTP_CF_ORG` / `existing.BTP_CF_SPACE`, then the
   configured org / space, then the current target from `status`.
2. `CLI apps --org <org> --space <space>` lists the apps bound to both a destination and a connectivity
   instance. Use `existing.BTP_APP`, else the configured tunnel app, if it is listed. Pick automatically when
   there is exactly one, ask when there are several.
3. If the chosen app has `sshEnabled: false`, tell the user to run `cf enable-ssh <app>` and `cf restart <app>`
   (or do it for them if they agree). If SSH is disabled for the whole space, a Space Manager has to run
   `cf allow-space-ssh <space>`.
4. If there is no app, explain that the proxy needs one, and offer to create `btp-tunnel`: a 64 MB app that only
   sleeps, which reuses or creates the `destination` and `connectivity` service instances with plan `lite`. This
   changes the user's BTP space, so get an explicit yes, then run `CLI create-tunnel --org <org> --space <space>`
   (a minute or two). If it fails on entitlements or quota, relay the message: the subaccount needs the
   entitlements Destination (lite), Connectivity (lite) and 64 MB of Cloud Foundry Runtime memory.

## Destinations

`CLI destinations --org <org> --space <space> --app <app>` lists the OnPremise HTTP destinations with `Name`,
`URL`, `sap-client`, `User`, `Authentication` and `supported`. Only those with `supported: true`
(`NoAuthentication` or `BasicAuthentication`) work, because the local proxy has no user token. Passwords are never
returned. If none is listed, the subaccount has no OnPremise destination the app can see: they're defined in the
BTP cockpit (Connectivity > Destinations) and need a connected Cloud Connector.

## Several environments

Each run sets up one environment, with its own API endpoint, org, space, app, destinations and ports. Environments
on different API endpoints can't be started at the same moment, because the scripts switch the global cf target:
start one, wait until its proxy is up, then start the other.

## Troubleshooting

- `npx` fails or is slow: the user can install once with
  `npm install -g @scodigis/generator-btp-on-premise-scripts`, and you call `btp-on-premise-setup` directly.
- "The cf CLI was not found in PATH": install it from https://github.com/cloudfoundry/cli.
- The token expired during the steps: log in again and repeat the failed command.
- More details and the manual first-time setup:
  https://github.com/DenisGalandSAP/generator-btp-on-premise-scripts#readme
