smf-nettune
===========

This script/manifest allows you to define parameters tunable by ipadm or ndd
which can then be applied and unapplied through Solaris SMF.

Based on the original concept by Hung-Sheng Tsao:

  https://blogs.oracle.com/hstsao/entry/smf_and_tcp_tuning

My enhancements place ipadm/ndd parameters directly into SMF instead of
hardcoded in the method as in the original.

Additional changes include the use of ipadm on Solaris 11 and illumos as
explained by Steffen Weiberle:

  https://blogs.oracle.com/stw/entry/solaris_11_express_network_tunables

If ipadm exists, it will be used over ndd. Be sure to check the output of
`ipadm show-prop tcp` on your system for property names, as they vary slightly
by release, and also differ from the ndd names.

The tuning values in this example originally came from the following page, but
were modified based on additional feedback from PSC network engineers for the
best 10GB performance:

  http://www.psc.edu/index.php/component/content/article/12-about/employment/641-tcp-tune#Solaris

To use ndd names not exposed through ipadm with ipadm, they must be prepended
with an underscore, however SMF property names may not begin with an
underscore. To set these tunables (e.g. `_cwnd_max`) create a property with the
leading underscore replaced with an uppercase `U` (e.g. (`Ucwnd_max`)).

preseed
-------

On SmartOS, it is not possible to set the property values since SMF is
essentially created from scratch on each boot. Because of this, you can create
a preseed file (by default, `/opt/custom/etc/nettune_preseed.conf` which will
populate the SMF values from this file. Its format is:

```
<device> <property_group> <property> <property_val>
```

E.g. for ipadm:

```
tcp send_buf integer 16777216
tcp recv_buf integer 16777216
tcp max_buf integer 33554432
tcp Ucwnd_max integer 33554432
```


usage
-----

1.  Copy the nettune script to somewhere appropriate for your environment and
    make sure it is executable.  By default, the SMF config expects to find it
    in `/lib/svc/method`.

2.  Copy network-nettune.xml to `/var/svc/manifest/site` (or on SmartOS, to
    `/opt/custom/smf`).  Edit the XML config file if you put nettune somewhere
    other than `/lib/svc/method`.

3.  Import the manifest:

    ```console
    # svccfg import /var/svc/manifest/site/network-nettune.xml
    ```

4.  Create property groups for the protocols (ipadm) or devices (ndd) you wish
    to tune:

    ```console
    # svccfg -s nettune addpg tcp application
    ```

    For ndd, the property group is translated to the device passed to ndd by
    prepending `/dev/` to the property group name.

5.  Set some parameters (ipadm):

    ```console
    # svccfg -s nettune setprop tcp/send_buf = integer: 16777216
    # svccfg -s nettune setprop tcp/recv_buf = integer: 16777216
    # svccfg -s nettune setprop tcp/max_buf = integer: 33554432
    # svccfg -s nettune setprop tcp/Ucwnd_max = integer: 33554432
    ```

    Or ndd:

    ```console
    # svccfg -s nettune setprop tcp/tcp_xmit_hiwat = integer: 16777216
    # svccfg -s nettune setprop tcp/tcp_recv_hiwat = integer: 16777216
    # svccfg -s nettune setprop tcp/tcp_max_buf = integer: 33554432
    # svccfg -s nettune setprop tcp/tcp_cwnd_max = integer: 33554432
    ```

6.  Create an SMF snapshot that includes your new values:

    ```console
    # svcadm refresh network/tune
    ```

7.  Check your current values before enabling:

    ```
    % ipadm show-prop -p send_buf tcp
    PROTO PROPERTY              PERM CURRENT      PERSISTENT   DEFAULT      POSSIBLE
    tcp   send_buf              rw   128000       --           128000       4096-1048576
    % ndd -get /dev/tcp tcp_xmit_hiwat
    128000
    ```

8.  Enable the service.

    ```console
    # svcadm enable network/tune
    ```

9.  Check the new values to make sure it worked:

    ```console
    % ipadm show-prop -p send_buf tcp
    PROTO PROPERTY              PERM CURRENT      PERSISTENT   DEFAULT      POSSIBLE
    tcp   send_buf              rw   16777216     --           128000       4096-1048576
    % ndd -get /dev/tcp tcp_xmit_hiwat
    16777216
    ```

10. You can disable the service to reset the tunables to their previous values,
    since the previous values are stored whenever the service is enabled.
    These previous values are stored in an SMF property group named like the
    one(s) you created, but with `_defaults` appended:

    ```console
    % /usr/bin/svcprop -p tcp_defaults network/tune
    tcp_defaults/send_buf integer 128000
    tcp_defaults/recv_buf integer 128000
    tcp_defaults/max_buf integer 1048576
    tcp_defaults/Ucwnd_max integer 1048576
    ```
