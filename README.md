# Zabbix Systemd Unit

Zabbix template for monitoring the state of selected Linux systemd units through active agent checks. It uses two custom UserParameters and is designed for both Zabbix agent and Zabbix agent 2.

Import [template_systemd_unit.yaml](template_systemd_unit.yaml), exported in **Zabbix 7.0** format. The template is named `Template Systemd Unit`, belongs to `Templates/Applications`. Compatibility with other Zabbix versions must be verified in the target environment.

## Requirements

- A Linux host running systemd, with `systemctl` available to the agent service account.
- Zabbix agent or Zabbix agent 2, configured for active checks.
- Connectivity from the agent to its Zabbix server or proxy, normally over TCP port 10051.
- The two UserParameters below, configured on every monitored host.

## Agent configuration

### Required UserParameters

Add these exact lines to the configuration file used by the installed agent:

```ini
UserParameter=check.args[*],echo "$1"
UserParameter=service.active[*],systemctl is-active "$1"
```

The required UserParameters are provided in Utilities/userparameter.conf.

You can either add these definitions to the main Zabbix agent configuration
file or copy userparameter.conf into the agent's configuration include
directory, typically:

- Zabbix agent: /etc/zabbix/zabbix_agentd.d/
- Zabbix agent 2: /etc/zabbix/zabbix_agent2.d/

Ensure the main configuration file contains an Include directive matching
the chosen directory, for example:

Include=/etc/zabbix/zabbix_agentd.d/\*.conf

| UserParameter       | Purpose                                                                                       | Example item key                                           |
| ------------------- | --------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `check.args[*]`     | Returns its argument as text. The discovery rule passes the configured unit list to this key. | `check.args["sshd.service%crond.service%rsyslog.service"]` |
| `service.active[*]` | Runs `systemctl is-active` for the specified unit and returns its textual state.              | `service.active["sshd.service"]`                           |

The `[*]` suffix enables arguments; Zabbix substitutes `$1` with the first item key argument. Keep `$1` literal when editing the configuration file. See the [Zabbix UserParameter documentation](https://www.zabbix.com/documentation/7.0/en/manual/config/items/userparameters).

### Active checks

Set the active check destination and host name in the same agent configuration:

```ini
ServerActive=zabbix.example.local
Hostname=LINUX-SERVER-01
```

Replace the example values with your environment settings. `Hostname` must match the technical host name in Zabbix. For a host monitored through a proxy, point `ServerActive` to that proxy.

Restart the installed agent service after saving the configuration. For typical package installations, use the applicable command:

```bash
# Zabbix agent
sudo systemctl restart zabbix-agent

# Zabbix agent 2
sudo systemctl restart zabbix-agent2
```

## Import and host setup

1. In the Zabbix frontend, open **Data collection → Templates → Import** and select `template_systemd_unit.yaml`.
2. Review the import preview and import the template.
3. Under **Data collection → Hosts**, create or open the Linux host and link `Template Systemd Unit`.
4. Override `{$SYSTEMD.NAME.SERVICE.MATCHES}` at host level with the units to monitor, then save.
5. Ensure the host is enabled and assigned to the correct server or proxy.
6. Allow the discovery rule to run, then check the discovered items under **Monitoring → Latest data** and alerts under **Monitoring → Problems**.

## Selecting units

| Macro                             | Default value                                |
| --------------------------------- | -------------------------------------------- |
| `{$SYSTEMD.NAME.SERVICE.MATCHES}` | `sshd.service%crond.service%rsyslog.service` |

Despite its name, this macro is an explicit list, not a regular expression. Separate unit names with `%` and use the full names available on each host. For example:

```text
nginx.service%postgresql.service%rsyslog.service
```

Service names can differ between distributions, so verify the names before copying the default list. Avoid duplicate entries and select units expected to remain active.

The template does not enumerate installed units. Every **30 minutes**, `Units Discovery` reads the macro through `check.args`, splits the returned text at `%`, trims surrounding whitespace, and ignores empty entries. It creates one `{#UNIT.NAME}` discovery entry for each remaining name. An empty list produces no discovery entries.

## Collected data and alerts

| Object                 | Configuration                                |
| ---------------------- | -------------------------------------------- |
| Discovery rule         | `Units Discovery`, active check, every `30m` |
| Item prototype         | `Unit {#UNIT.NAME}: Status`                  |
| Item key               | `service.active["{#UNIT.NAME}"]`             |
| Collection             | Active check, every `5m`, text value         |
| Trigger prototype      | `Unit {#UNIT.NAME}: Not running`             |
| Trigger severity       | `HIGH`                                       |
| Manual problem closure | Enabled                                      |

The item stores the state returned by `systemctl is-active`. The trigger compares that state with the literal string `active`; it does not inspect application health or test whether the service is enabled at boot.

The current trigger expression is:

```text
last(/Template Systemd Unit/service.active["{#UNIT.NAME}"],#2)<>"active"
```

## Verification and troubleshooting

On the monitored Linux host, verify that the agent account can run the command for a configured unit. Adapt the account and unit name as needed:

```bash
sudo -u zabbix systemctl is-active sshd.service
```

For a unit expected to be active, the output should be `active`. Investigate other results with `systemctl status` and the relevant service logs.

- **Unsupported item key:** confirm that both UserParameters are loaded by the running agent and that the service was restarted after configuration changes.
- **No discovered units:** verify the host macro value, the discovery rule status, and its preprocessing errors. Discovery runs every 30 minutes; it does not scan the system for unit names.
- **No active check data:** verify `ServerActive`, `Hostname`, server/proxy assignment, network connectivity, and the agent log.
- **Unexpected unit state:** check the exact unit name, availability of `systemctl`, and access to systemd from the agent service account.
- **Alert differs from the latest state:** inspect the two most recent values; the current trigger uses the older one through `last(...,#2)`.
- **No alert when the agent stops reporting:** this template has no `nodata()` or agent availability trigger. Monitor host and agent availability separately.
