# Running

The drachtio-server executable built from source code is simply named `drachtio`.  By default, the build process installs it in `/usr/local/bin/drachtio`.

On systemd distributions, drachtio can be installed as a systemd service -- the [ansible role](https://github.com/davehorton/ansible-role-drachtio) does this automatically; on a build from source the systemd script [can be found here](https://github.com/davehorton/drachtio-server/blob/develop/drachtio.service).  The following options are supported:

```bash
$ systemctl [start|stop|restart|status] drachtio
```

Command-line options for the `drachtio` executable will be covered in the following section.