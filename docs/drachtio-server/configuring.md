# Configuring

The drachtio process can either be configured via command-line parameters, environment variables, a configuration file -- or a combination of all three.  The order of precedence of configuration settings is as follows:
* command-line parameters, if provided, always take precedence
* otherwise environment variables, if provided, take precedence
* otherwise configuration file settings are used.

By default, the server will look for a configuration file at `/etc/drachtio.conf.xml`.  This can be changed by passing the configuration file path as a command line argument `-f`; e.g., `drachtio -f /tmp/drachtio.conf.xml`

As the name suggests, the configuration file is in XML format.  The structure of the file is described below, and [a heavily commented version of the file can be found here](https://github.com/davehorton/drachtio-server/blob/develop/drachtio.conf.xml) that provides additional detail to the summary provided below. 

### drachtio.conf.xml

A drachtio configuration file has the following high-level structure:
```xml
<drachtio>
  <admin/>
  <request-handlers/>
  <sip/>
  <cdrs/>
  <monitoring/>
  <logging/>
</drachtio>
```
Each section is described below, along with the command line parameters and environemnt variables that can be used to configure the same settings.

#### admin section
The **admin** section is required and specifies how the drachtio server will listen for incoming connections from drachtio applications. The information includes the tcp port to listen on, the address(es) to listen on (0.0.0.0 means all available interfaces), the shared secret that is used for authentication, and whether tcp keep alives will be sent on these connections.

Note that as of release 0.8.0, there is also an option to use tls encryption on connections.  For inbound connections, this is specified by providing a 'tls-port' option.  The server can be configured to handle either, or both, tcp and tls connections.
```xml
<admin port="9022" tls-port="9023" secret="cymru" tcp-keepalive-interval="30">0.0.0.0</admin>
```
or, using command-line parameters:
```bash
drachtio --port 9022 --tls-port 9023 --secret cymru --tcp-keepalive-interval 30
```
or, using environment variables
```bash
DRACHTIO_ADMIN_TCP_PORT=9022 \
DRACHTIO_ADMIN_TLS_PORT=9023 \
DRACHTIO_TCP_KEEPALIVE_INTERVAL=30 \
DRACHTIO_SECRET=cymru drachtio
```

Note that by default, tcp keepalives are enabled with an interval of 45 seconds.  The value can be changed, as above, or disabled by setting it to a value of zero.

#### request-handlers section
The **request-handlers** section is optional and configures the drachtio process to establish [outbound connections](/docs#outbound-connections) to drachtio servers for some or all SIP methods instead of **inbound connections**.

The `<request-handlers>` element can have zero or more child `<request-handler>` elements. Each `<request-handlers>` defines a specific SIP method (or `*` to wildcard all methods) and an http(s) web callback to invoke when a new request of the specified method type arrives.  It is the responsibility of the user-supplied web callback to return information in an HTTP 200 OK response indicating how to route the call.

```xml
<request-handlers>
    <request-handler sip-method="INVITE" http-method="GET" verify-peer="false">https://38.187.89.96:8080</request-handler>
</request-handlers>
```
With the configuration above in place, when the drachtio server receives a new incoming INVITE request, it will send an HTTP GET to the URL above, with HTTP query arguments
* `method`: the SIP method of the request
* `domain`: the SIP domain in the Request-URI
* `protocol`: the transport protocol used (e.g. 'udp', 'tcp', 'wss', etc)
* `source_address`: the IP address of the sender
* `fromUser`: the user part of the uri in the From header
* `toUser`: the user part of the uri in the To header
* `uriUser`: the user part of the uri in the Request-URI
* `contentType`: the Content-Type header in the request, if any
* `uri`: the full Request-URI

> Note: either HTTP or HTTPS URIs are supported.  If using self-signed certificates with HTTPS, set `verify-peer` to false, as above.

> Note: setting `http-method="POST"` will cause an HTTP POST top be sent to the user-supplied web callback.  All of the information supplied below will be provided (e.g. query args) but additionally the body of the request will have a full copy of the incoming SIP request message.  This is useful in more complex routing scenarios which may, for instance, depend on examining specific values in the SIP headers of the incoming message.

An example HTTP URL that gets sent out looks like this:
```
 http://38.187.89.96:8080/?method=INVITE&domain=server-01.drachtio.org&protocol=udp&source_address=10.132.0.29&fromUser=%2b15083084809&toUser=calltest&uriUser=r-ee78299f-2f85-4d92-97ab-24f1d11e2b69&contentType=application%2fsdp&uri=sip%3ar-ee78299f-2f85-4d92-97ab-24f1d11e2b69%server-01.drachtio.org%3bdst%3d%2b15083084809%2540139.59.165.83%3a5060&dst=%2b15083084809%2540139.59.165.83%3a5060
 ```

Based on the information above provided in the HTTP request, the user-supplied callback is responsible for indicating one of the following actions in a JSON body of the HTTP 200 OK response:
* return a non-success response to the request
* proxy the request
* redirect the request (valid for INVITE only)
* route the request to a specified drachtio application

The first three actions completely disposition the incoming SIP request -- i.e. no further interaction with a drachtio application occurs.  

The final action (route to an application) causes the drachtio server to establish an outbound tcp connection to a drachtio application listening on a specified port, which then receives and processes the request normally (e.g. in a `srf.invite((req, res)))` or equivalent).

Note that as of release 0.8.0, it is possible route to a drachtio application over an outbound connection using tls.  This is specified by appending a `transport` attribute to the uri and specifying 'tls', e.g. `uri:myapp.example.com;transport=tls`.

Example JSON responses for each of the above action are illustrated below (note: a response should include only one of the JSON payloads below):

```js
// this would reject the call with a "503 Max Calls Exceeded" response
// note: reason is optional
{
  "reject":
  "data": {
      "status": 503,
      "reason": "Max Calls Exceeded"
  }
}

// this redirects the call to the address specified.
// the Contact header of the response will be populated accordingly
{
  "redirect":
  "data": {
      "contacts": [
          "sip:foo@bar.com"
      ]
  }
}

// this proxies the call accordingly
{
  "proxy":
  "data": {
      "destination": [
          "sip:foo@bar.com"
      ],
      recordRoute: true
  }
}

// this causes the request to be delivered to a drachtio app for further processing.
// the drachtio app must be listening on the uri provided; i.e an outbound connection.
{
  "route":
  "data": {
      "uri": "call-recording.default.svc.cluster.local:4000"
  }
}

// this causes the request to be delivered to a drachtio app for further processing.
// the drachtio app must be using tagged inbound connections.
{
  "route":
  "data": {
      "tag": "conferencing-app"
  }
}
```
> Note: the last stanza above applies to using tagged inbound connections.  For more details, [see here](https://drachtio.org/blog/introducing-tagged-inbound-connections/)

A request handler for all incoming SIP requests can be configured via the command-line as well:
```bash
drachtio --http-handler "http://38.187.89.96:8080" --http-method "GET"
```


#### sip section

The `<sip>` section defines which addresses and ports the SIP stack will listen on, which protocols will be supported, where to find (if necessary) SSL certificates, and other SIP options.

##### contacts
The drachtio server can listen on multiple interfaces/addresses for SIP traffic. These are defined in a `<contacts>` element that has child `<contact>` elements for each SIP endpoint.  Examples of possible configuration are shown below.

```xml
<!-- listen on all addresses, default port 5060 for udp and tcp protocols -->
<sip>
  <contacts>
    <contact>sip:*;transport=udp,tcp</contact>
  </contacts>
</sip>
```
```xml
<!-- listen on ports 5060 and 5080 -->
<contacts>
  <contact>sip:*;transport=udp,tcp</contact>
  <contact>sip:*:5080;transport=udp,tcp</contact>
</contacts>
```
```xml
<!-- listen for secure websockets on specific address and port -->
<contacts>
  <contact>sip:192.168.100.23:443;transport=wss</contact>
</contacts>
```
Additionally, if the SIP server has been assigned an external address that should be used in the SIP signaling, this should be specified as follows:
```xml
<contacts>
  <contact external-ip="35.195.28.194">sip:10.132.0.22;transport=udp,tcp</contact>
</contacts>
```
This will cause the drachtio server to advertise its address as `35.195.28.194` in Contact and Via headers, even though its local assigned IP address is `10.132.0.22`.

Furthermore, if the drachtio server has an assigned DNS name, this should be configured as well so that it can detect when the Request-URI of an incoming SIP request is referring to the local host when the DNS name appears in the host portion.
```xml
<contacts>
  <contact dns-names="server01.drachtio.org" external-ip="35.195.28.194">sip:10.132.0.22;transport=udp,tcp</contact>
</contacts>
```
> Note: multiple DNS names can be provided in comma-separated format.

SIP contacts can be supplied via command line as follows
```
drachtio --contact "sip:10.132.0.22;transport=udp" --external-ip 35.195.28.194
```
##### timers
The [SIP spec contains definitions for timers](https://tools.ietf.org/html/rfc3261#page-265) governing retransmissions of SIP requests and the like.  Generally, there is no need to modify the setting for these timers, but if desired this can be done as follows:
```xml
<timers>
  <t1>500</t1>
  <t2>4000</t2>
  <t4>5000</t4>
  <t1x64>32000</t1x64>
</timers>
```
> Note: values are in milliseconds.  The example above actually sets the timers to their defined default values, so if you are using this section you would like be setting them to some other values.  You only need to specify those timers that you want to adjust from their default values.

##### tls
If you are using either TLS or WSS as a transport, then you must specify where the associated tls certificates are stored on the server.

Additionally, when using tls on admin connections from applications, you must specify a dhparam file that contains the Diffie-Hellman (dh) parameters.  (This is not required if you are only using TLS to secure SIP connections)

```xml
<tls>
  <key-file>/etc/letsencrypt/live/yourdomain/privkey.pem</key-file>
  <cert-file>/etc/letsencrypt/live/yourdomain/cert.pem</cert-file>
  <chain-file>/etc/letsencrypt/live/yourdomain/chain.pem</chain-file>
  <dh-param>/var/local/private/dh4096.pem</dh-param>
</tls>
```
or, via command-line
```
drachtio --key-file <keyfile> --cert-file <certfile> --chain-file <chainfile> --dh-param <dhparamfile
```

##### outbound-proxy
This causes all outbound requests to be sent through an outbound proxy
```xml
<outbound-proxy>sip:10.10.10.1</outbound-proxy>
```

##### spammers
The drachtio server can examine the Contact, To, and From headers for distinctive signatures that indicate the request was sent from a spam source.  If a spammer is detected, the message can either be rejected or silently discarded.
```xml
<spammers action="reject" action="discard">
  <header name="User-Agent">
    <value>sip-cli</value>
    <value>sipcli</value>
    <value>friendly-scanner</value>
  </header>
  <header name="To">
    <value>sipvicious</value>
  </header>
</spammers>
```

##### capture-server
The drachtio server can be configured to send to [Homer](http://www.sipcapture.org/) using the [HEP](https://github.com/sipcapture/HEP) protocol.

```xml
<capture-server port="9060" hep-version="3" id="101">127.0.0.1</capture-server>
```
or, via command line
```
drachtio --homer "127.0.0.1:9060" --homer-id 101
```
or, via environment variables:
```
DRACHTIO_HOMER_ADDRESS=127.0.0.1 \
DRACHTIO_HOMER_PORT=9060 \
DRACHTIO_HOMER_ID=101 drachtio
```

##### udp-mtu
*Added in version 0.7.3-rc2*

sofia-sip has an annoying feature where it forces an outbound request to go out TCP if the packet size exceeds a specific threshold (usually 1300 bytes). Tis configuration setting allows users to increase this threshold to an arbitrary value.

```xml
<udp-mtu>4096</udp-mtu>
```
or, via command line
```
drachtio --mtu 4096
```
or, via environment variables
```
DRACHTIO_UDP_MTU=4096 drachtio
```
#### monitoring section
drachtio supports [prometheus](https://prometheus.io) monitoring by optionally exposing a /metrics endpoint.  [See here](https://github.com/davehorton/drachtio-server/blob/develop/docs/prometheus.md) for a list of the metrics provided
```xml
<monitoring>
  <prometheus port="9090">127.0.0.1</prometheus>
</monitoring>
```
> Note: if the address is not provided, the `/metrics` endpoints will be available on all interfaces (e.g. 0.0.0.0).

or, via command line:
```
drachtio --prometheus-scrape-port "9090"
# above implies 0.0.0.0:9090, we can be more explicit
drachtio --prometheus-scrape-port "10.0.1.5:9090"
```
or, via environment variables
```
DRACHTIO_PROMETHEUS_SCRAPE_PORT=9090 drachtio
# or
DRACHTIO_PROMETHEUS_SCRAPE_PORT=10.0.1.5:9090 drachtio

```
#### logging section

The `<logging>` section defines where drachtio server will send logging information, including sip traces.

Logging destinations include the console, a named log file on the server, or syslog.  Any or all of them may be used at one time.

##### console

To log output to the console simply include a `<console\>` child element.

##### syslog

To send log output to a syslog server via UDP, specify the following:
```xml
<syslog>
  <address>127.0.0.1</address>
  <port>514</port>
  <facility>local6</facility>
</syslog>
```

##### log file

To send log output to a log file on the server, specify the following:
```xml
 <file>
  <name>/var/log/drachtio/drachtio.log</name>
  <archive>/var/log/drachtio/archive</archive>
  <size>50</size> 
  <maxSize>100</maxSize>
  <minSize>2000</minSize>
  <auto-flush>true</auto-flush>
</file>
```
The options are as follows:
* `name`: path to the log file
* `archive`: path a directory where older log files were be archived
* `size`: the size (in MB) at which the log file is truncated
* `maxSize`: the max size (in MB) of archived files to keep
* `minSize`: the minimum freespace (in MB) on the filesytem to maintain when archiving
* `auto-flush`: if true, log information is written immediately to disk; otherwise log file is buffered and written intermittently (slightly better performance)

##### loglevel

The overall system log level: 'notice', 'warning', 'error', 'info, or 'debug'.

```xml
<loglevel>info</loglevel>
```

> Note: 'info' is the recommended log level for production systems.  At this log level you will get sip traces, which are useful for debugging.

##### sofia-loglevel

The drachtio server uses the [sofia](https://github.com/davehorton/sofia-sip) library internally.  The log level for this library can be set from 0 (minimal) to 9 (extensive).

```xml
<sofia-loglevel>3</sofia-loglevel>
```


### command-line arguments

The `drachtio` executable can accept command-line arguments that specify some configuration parameters.  If provided, the command-line configuration parameters take preference over those specified in the configuration file.

The supported drachtio command-line arguments are:
* `--daemon` detach from the console and run as a daemon process.  Note: when running as a systemd service, this parameter is not necessary.
* `--noconfig` ignore any logging configuration in the configuration file
* `--file|-f filename` read configuration from specified file rather that `/etc/drachtio.conf.xml`
* `--user|-u user` run as the named user rather than root
* `--port|-p port` listen for tcp admin connections on the named port
* `--tls-port` listen for tls admin connections on the named port. &nbsp;*Added in version 0.8.0-rc1*.
* `--contact|-c` specifies a listening address/port/protocol.  Multiple instances of this parameter may be provided
* `--external-ip ip-address` specifies an external address that the drachtio server should advertise in the SIP signaling. This parameter applies to the `--contact` parameter that it follows in the command line.
* `dns-name name` a dns name that refer to the local server. This parameter applies to the `--contact` parameter that it follows in the command line.
* `http-handler url` an HTTP URL of a web callback that will be invoked for all new incoming requests.  Setting this parameter turns on outbound connections for all SIP request types.
* `http-method` either 'GET' or 'PUT'
* `--loglevel level` the overall log level to set
* `--sofia-loglevel level` the log level of the sofia library
* `--stdout` write log output to console
* `--homer ip-address:port` ip address of homer capture server to send to. HEP3  and udp transport will be used
* `--homer-id id` id to use to represent this server when sending messages to homer
* `--version` print the drachtio server version to console and exit.
* `--mtu` specifies a message size, in bytes, for requests such that when outgoing requests exceed this threshold use of tcp is forced (this overrides the default sofia stack setting for the same). &nbsp;*Added in version 0.7.3-rc2*.
* `--dh-param` dhparam file used for inbound tls admin connections. &nbsp;*Added in version 0.8.0-rc1*.






