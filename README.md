# grafana-monit

The scripts in this repository are wrappers for the Grafana OnCall Web Hooks which will
open and close Grafana OnCall incidents from the command line. These scripts are
designed to help you integrate monit with Grafana OnCall.

This is based off of https://github.com/pinterest/pagerduty-monit and
https://github.com/etheriau/opsgenie-monit

## Usage ##

`grafana-trigger <service-name>`

`grafana-resolve <service-name>`

### Example ###

`grafana-trigger nginx` on the server "webserver1.example.com"
will trigger a OpsGenie incident with the subject "nginx failed on
webserver1".

## Prerequisites ##

Requires requests library for Python. You can run `pip3 install requests`.

## To configure Grafana Oncall ##

To configure Grafana OnCall's Integration section:

* Click New Integration
* Select WebHook
* Enter a Name and Assign a Team
* Select Create Integration
* To the left of HTTP Endpoint, select Copy
* Replace the `YOUR_ENDPOINT` in `API_ENDPOINT = "YOUR_ENDPOINT"` with your key in both scripts
* In the Templates section, click Edit
  * Grouping to ```{{ payload.get("message", "") }}```
  * Change each Title to ```{{ payload.get("message", "") }}```
  * Change each Message to ```{{ payload.get("description", "") }}```

## To configure Monit ##

To configure Monit to trigger an event with this script, save the script in a location
like /etc/monit, make sure it's executable, and use an action like this in your monitrc file:

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

By default, the script is designed to fire new alerts only if one hasn't been recently fired.  This is done via a file. 
You can disable this capability via `--dontUseFiles`.

Also, by default, the alert will not refire after 4 hours.  You can enable this feature via `--realert`.

Note that for `--dontUseFiles` and the refire after 4 hours, by default, the configuration de-dupes based on the title,
so you will need to edit the template's Grouping if you really want to use this.