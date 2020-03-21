# Advanced Topics

## Outbound connections

The examples so far have illustrated **inbound connections**; that is, a drachtio application establishing a tcp connection to a drachtio server.  These are created by calling [Srf#connect](/docs/api#Srf+connect):
```js
const Srf = require('drachtio-srf');
const srf = new Srf();

// example of creating inbound connections
srf.connect({
  host: '192.168.1.100',
  port: 9022, 
  secret: 'cymru'
});

srf.on('connect', (hp) => console.log(`connected to drachtio listening on ${hp}`));
srf.on('error', (err) => console.log(`error connecting: ${err}`));

srf.invite((req, res) => {..});
```
An inbound connection is intended to be a **long lasting** connection: the application connects when the application starts, and that connection is then used to transmit all SIP events and commands as long as the application is running.  

> Note: If the connection between the drachtio client and the server is interrupted, it will be automatically reestablished as long as the application has installed a listener for the 'error' event on the Srf instance.

Inbound connections are generally most useful in scenarios when a drachtio server is single-purposed, meaning all SIP requests are handled by a single application.  For example, if a drachtio server is specifically purposed to be a SIP proxy, and all incoming calls are treated by the same application logic, then an inbound connection between the drachtio application and server would probably be preferred.

However, it is also possible for the connections between the drachtio application and the server to be reversed: that is, the drachtio server establishes the connection to a drachtio server on a per-call (more specifically, a per SIP request) basis.  This is called an **outbound connection**, and it requires two things:
1. The drachtio server configuration file must include a [request-handler](/docs/drachtio-server#request-handlers-section) xml section that maps a specific SIP method type to an HTTP web callback, and
2. The drachtio application must call the [Srf#listen](/docs/api#Srf+listen) method, which causes it to listen for connections from a drachtio server rather than initiate them.

The sequence of events when outbound connections have been enabled are as follows:
1. The drachtio application starts, and begins listening on a specific IP address and port for connections from drachtio servers.
2. An incoming INVITE request (for example) is received by a drachtio server.
3. Because the drachtio server has been configured with a web callback for INVITE request types, an HTTP GET request is made to the web callback.  Information about the incoming call is passed to the web callback via url parameters in the request.
4. The web callback -- which is a user-supplied web app -- returns a JSON body indicating the ip address or dns name and tcp port where the drachtio application is listening.
5. The drachtio server retrieves the ip address and port from the response to the web callback and establishes a tcp connection to the drachtio application listening on that address:port.
6. The INVITE information is sent to the drachtio application over this new connection, and the standard drachtio middleware is invoked; e.g. `srf.invite((req, res))`.

```js
const Srf = require('drachtio-srf');
const srf = new Srf();

// example of listening for outbound connections
srf.listen({
  port: 3001, 
  secret: 'cymru'
});

srf.invite((req, res) => {..});
```

From the standpoint of the drachtio application you would write, the code is almost exactly the same other than the call to [Srf#listen](/docs/api#Srf+listen) instead of [Srf#connect](/docs/api#Srf+connect) and one other matter related to eventually releasing the connection, which we will describe shortly.

### Mixing inbound and outbound connections

Is it possible to mix inbound and outbound connections?  

Sort of.  Here are the limitations:

1. A drachtio server can have both inbound and outbound connections, but a given SIP method type (e.g. INVITE) will exclusively use one or the other.  Specifically, if there is a `<request-handler>` element with a `sip-method` property set to either the method name or `*`, then outbound connections will be used for all incoming SIP requests of that method type; otherwise inbound connections will be used.
2. A single `Srf` instance must exclusively use only inbound or outbound connections; that is to say that it must call *either* [Srf#connect](/docs/api#Srf+connect) *or* [Srf#listen](/docs/api#Srf+listen) but not both.  A single application that wants to use both must create two (or more) Srf instances, or alternatively the functionality can be split into multiple applications.

### Terminating outbound connections

We mentioned above that inbound connections are long-lasting.

Outbound connections are not.  

An outbound connection is established when a specific SIP request arrives, and it is intended to last only until the application has determined that all logic related to that request has been performed. From a practical standpoint, since each new arriving request spawns a new tcp connection, it is important that connections are destroyed when the application logic is complete, so that we don't exhaust file descriptor or other resources on the server.

Because the determination of "when all application logic has complete" is, by definition, something that only the application can know, we require the application to destroy the connection via an explicit call to [Srf#endSession](/docs/api#Srf+endSession) when it is no longer needed.  Typically, an application will call this method when all SIP dialogs or transactions associated with or emanating from the initial SIP request have been destroyed.  

In a simple example of a UAS app connecting an incoming call, for instance, when the BYE that terminates the call is sent or received it would be appropriate to call [Srf#endSession](/docs/api#Srf+endSession).

> This method call is a 'no-op' (does nothing) when called on an inbound connection, so it is safe to call in code that may be dealing with an outbound or inbound connection.

### When to use outbound connections

There are two primary scenarios in which to use outbound connections:

1. When a single drachtio server is going to handle calls controlled by multiple different types of applications.  In this scenario, it can be useful to have a web callback examine the incoming calls and distribute them appropriately to the different drachtio applications based on per-call information.  For example, if we wanted to stand up a drachtio server that multiple customers could utilize (i.e. multi-tenant situation) then outbound connections would allow us to have many different customer applications controlling calls on that server, each applying their own logic.
2. When drachtio applications are going to run in a containerized cluster environment such as Kubernetes, outbound connections can be useful.  In this environment, it can be useful to create a Kubernetes Service for the drachtio cluster, and then use outbound connections to route incoming calls to the public address of the Kubernetes Service which is backed by a cluster of drachtio applications running in Kubernetes pods.  

In general, outbound connections can make it easier to independently scale drachtio servers and groups of drachtio applications, since you do not need to explicitly "tie" drachtio applications to specific servers.

For more information on configuring drachtio server for outbound connections refer to the [drachtio server configuration documentation](/docs/drachtio-server#request-handlers-section).

## Using TLS to encrypt messages between application and server

As of drachtio server release 0.8.0-rc1 and drachtio-srf release 4.4.0, it is possible to encrypt the messages between the drachtio server and your application.  This may be useful in situations where applications are running remotely and you prefer to encrypt the control messages as they pass through intervening networks.  Both inbound and outbound connections can use TLS encryption, though the configuration steps are different as described below.

### Securing inbound connections using TLS

To use TLS on inbound connections, simply configure the drachtio server to listen on a specific port for TLS traffic, in addition to (or in place of) TCP traffic.  For example:
```
<admin port="9022" tls-port="9023" secret="cymru">127.0.0.1</admin>
```
would cause the server to listen for tcp connections on port 9022 and tls connections on port 9023.

You can also specify the port on the command line:
```
drachtio --tls-port 9023
```
In addition to specifying a port to listen for tls traffic, you must specify minimally a server key, a certificate, and a dhparam file.  These are specified in the 'tls' section of the config file:
```
<tls>
    <key-file>/etc/letsencrypt/live/example.org/privkey.pem</key-file>
    <cert-file>/etc/letsencrypt/live/example.org/cert.pem</cert-file>
    <chain-file>/etc/letsencrypt/live/example.org/chain.pem</chain-file>
    <dh-param>/var/local/private/dh4096.pem</dh-param>
</tls>
```
Of course, these can also be specified via the command line as well:
```
drachtio --dh-param /var/local/private/dh4096.pem \
  --cert-file /etc/letsencrypt/live/example.org/cert.pem \
  --chain-file /etc/letsencrypt/live/example.org/chain.pem \
  --key-file /etc/letsencrypt/live/example.org/privkey.pem
```
#### drachtio-srf app configuration
On the client side, when connecting to a TLS port the [Srf#connect](/api#srf-connect) function call must include a 'tls' object parameter in the options passed:
```
    this.srf.connect({
      host: '127.0.0.1',
      port: 9023, 
      tls: {
        rejectUnauthorized: false
      }
    });
```
Any of the node.js tls options that can be passed to [tls.createSecureContext](https://nodejs.org/api/tls.html#tls_tls_createsecurecontext_options) can be passed. Even if you do not need to include any options, you must still include an empty object as the `opts.tls` param in order to signal the underlying library that you wish to establish a TLS connection.

#### Using self-signed certificate on the server
If you are using a self-signed certificate on the server, then you must load that same certificate on the client, as below:
```
    this.srf.connect({
      host: '127.0.0.1',
      port: 9023, 
      tls: {
        ca: fs.readFileSync('server.crt'),
        rejectUnauthorized: false
      }
    });
```
### Securing outbound connections using TLS
To use TLS to secure outbound connections, there is no specific configuration needed on the server.  You just need your http request handler to return a uri with a `transport=tls` parameter, e.g.:
```
{"uri": "10.32.100.2:808;transport=tls"}
```
#### drachtio-srf configuration
On the application side, to listen for TLS connections you will need to modify the [Srf#listen](/api#srf-listen) function to pass tls options.  Minimally, you must specify a private key and certificate.
```
  srf.listen({
    port: 8080,
    tls: {
      key: fs.readFileSync('server.key'),
      cert: fs.readFileSync('server.crt'),
      rejectUnauthorized: false      
    }
  });
```
