# Scodigis plugins for Claude Code

A Claude Code plugin marketplace for SAP work: SAP BTP, SAPUI5 and ABAP development on on-premise systems.

Add the marketplace once, then install the plugins you need:

```text
/plugin marketplace add DenisGalandSAP/scodigis-claude-plugins
/plugin install btp-localhost-on-premise-access@scodigis
```

`/plugin marketplace update scodigis` brings in new versions.

## Plugins

| Plugin | What it does |
|---|---|
| [btp-localhost-on-premise-access](plugins/btp-localhost-on-premise-access) | Reach an on-premise SAP system from your own machine through an SAP BTP destination and the Cloud Connector. Run and deploy SAPUI5 and SAP Fiori elements apps, or work in ABAP with ABAP Development Tools (VS Code, Eclipse) and ADT MCP servers. Windows, macOS and Linux. |

### btp-localhost-on-premise-access

Many on-premise ABAP systems can only be reached through a BTP destination: the Cloud Connector connects them to
a subaccount, and nothing on your laptop can open them directly. The plugin sets up a small local proxy that goes
through a `cf ssh` tunnel to the connectivity proxy of a Cloud Foundry app, adds the destination's credentials,
and forwards your requests to the system. Your app, `fiori deploy`, ADT or an MCP server then talk to
`http://127.0.0.1:<port>` as if the system were local.

It has three commands:

- `/btp-localhost-on-premise-access:setup-ui5 [env]` sets up a SAPUI5 or SAP Fiori elements app (or library) to
  run and deploy through a destination.
- `/btp-localhost-on-premise-access:setup-abap [env]` sets up a folder for ABAP development on a system: the proxy
  with an https listener for ADT, a localhost certificate that ADT trusts, and optionally an ADT MCP server so
  Claude can read or change ABAP objects.
- `/btp-localhost-on-premise-access:run [env] [start | proxy | deploy | deploy-test]` checks the Cloud Foundry
  login and starts the app, the proxy or a deployment in the background.

Claude never asks for a password in the chat. The Cloud Foundry password stays in an environment variable you set
in your own terminal; the plugin only knows the variable's name. When the session expires, the scripts log in
again on their own, so a proxy started in the background keeps working. A rejected password is remembered and
not tried again until you change it, so a wrong or rotated password doesn't end up locking your account.

You can save your usual org, space, user and password variable in the plugin configuration (`/plugin`, Installed,
then Configure) so the setup commands don't ask for them each time.

The plugin drives the command-line tool of the npm package
[`@scodigis/generator-btp-on-premise-scripts`](https://github.com/DenisGalandSAP/generator-btp-on-premise-scripts),
which is also available as the "BTP On-Premise Access" generator in the SAP Fiori tools Template Wizard. Details,
requirements and data handling are in the [plugin's README](plugins/btp-localhost-on-premise-access).

## Requirements

- Node.js 22 or later, and the [cf CLI](https://github.com/cloudfoundry/cli) v8
- For ADT: OpenSSL (on Windows it comes with Git for Windows) and `/sap/bc/adt` exposed in the Cloud Connector

## Support

Open an issue in this repository. For problems with the generated scripts or the command-line tool, use the
[generator's repository](https://github.com/DenisGalandSAP/generator-btp-on-premise-scripts/issues).

## License

[MIT](LICENSE)
