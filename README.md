# grafana-monit

The scripts in this repository are wrappers for the Grafana OnCall webhook integration, which open and resolve Grafana OnCall alerts from the command line. They are designed to help you integrate monit with Grafana OnCall.

On Grafana Cloud, OnCall is now part of Grafana Cloud IRM, found under **Alerts & IRM**; the webhook integration these scripts post to works the same way there. The self-hosted open-source Grafana OnCall was [archived on March 24, 2026](https://grafana.com/docs/oncall/latest/set-up/open-source/): an existing install keeps working, but it is no longer developed.

This is based off of https://github.com/pinterest/pagerduty-monit and https://github.com/etheriau/opsgenie-monit

## Usage ##

`grafana-trigger <service-name> [--description <description>] [--dontUseFiles] [--realert]`

`grafana-resolve <service-name> [--dontUseFiles] [--force]`

### Example ###

`grafana-trigger nginx` on the server "webserver1.example.com" will open a Grafana OnCall alert with the title "nginx failed on webserver1.example.com". `grafana-resolve nginx` on the same server resolves it.

Both scripts exit with status 1 when they did not reach Grafana (an error, a timeout after 30 seconds or a response other than 2xx) or had nothing to do.

## Prerequisites ##

Requires Python 3 and the requests library. You can run `pip3 install requests`.

## To configure Grafana OnCall ##

To configure Grafana OnCall's Integrations section (the screens differ slightly between versions):

* Click New Integration
* Select Webhook
* Enter a Name and Assign a Team
* Select Create Integration
* To the left of HTTP Endpoint, select Copy, and give the scripts that URL (see **Configuring the endpoint** below)
* In the Templates section, click Edit
  * Grouping to ```{{ payload.get("message", "") }}```
  * Auto-resolve condition to ```{{ payload.get("state", "").upper() == "OK" }}```
  * Change each Title to ```{{ payload.get("message", "") }}```
  * Change each Message to ```{{ payload.get("description", "") }}```

The grouping and auto-resolve templates are what make resolving work. `grafana-resolve` posts the same `message` as the trigger with `"state": "OK"`, so the grouping puts it in the alert group the trigger opened, and the auto-resolve condition resolves that group.

## Configuring the endpoint ##

The scripts look for the integration's URL in this order and use the first one set:

1. `API_ENDPOINT` at the top of each script, if you replace `YOUR_API_ENDPOINT` with your URL.
2. The `GRAFANA_ONCALL_ENDPOINT` environment variable.
3. `/etc/monit/grafana-oncall.conf`, holding the URL on a line of its own (lines starting with `#` are ignored).

The file is the easiest to manage across many hosts, since the scripts can then be installed unchanged and upgraded by copying them over.

## Where alerts are remembered ##

So that a failure that persists does not page you on every check, `grafana-trigger` records each alert it opens as a file, and does nothing while that file exists. `grafana-resolve` resolves only when there is such a file, and deletes it once Grafana accepts the resolve. That makes it safe to call `grafana-resolve` on every healthy check.

The files live in `/var/lib/grafana-monit`, which survives a reboot. Set `GRAFANA_MONIT_STATE_DIR` to use another folder. If the folder cannot be created (for example, the script is not running as root), the scripts use `/tmp` instead. Versions before this one always used `/tmp`, and the scripts still find a file left there, so an alert opened before an upgrade is resolved normally.

If the file is lost while an alert is open, such as a container rebuilt without a volume for the state folder, `grafana-resolve` will not resolve that alert by itself. Run `grafana-resolve <service-name> --force` once, or resolve it in Grafana.

## To configure Monit ##

To configure Monit to trigger an event with this script, save the script in a location like /etc/monit, make sure it's executable, and use an action like this in your monitrc file:

    exec "/etc/monit/grafana-trigger <service-name>"

Example monit stanza to trigger an incident if nginx is not present:

    check process nginx with pidfile /var/run/nginx.pid
        if does not exist for 3 cycles
            then exec "/etc/monit/grafana-trigger nginx"
        else if passed for 3 cycles
            then exec "/etc/monit/grafana-resolve nginx"


## Other Uses ##

While the script is written for monit specifically, we use it for various alerts, especially for repeated triggers.  The use is:

```
    /etc/monit/grafana-trigger <summary> --description <description>
```

By default, the script is designed to fire new alerts only if one hasn't been recently fired, using the files described in **Where alerts are remembered**. You can disable this capability via `--dontUseFiles`; resolve such an alert with `grafana-resolve <summary> --dontUseFiles`, which resolves without looking for a file.

Also, by default, an alert does not fire again while it is open. With `--realert`, it fires again once it has been open for 4 hours.

Note that for `--dontUseFiles` and the refire after 4 hours, by default, the configuration de-dupes based on the title, so you will need to edit the template's Grouping if you really want to use this.
