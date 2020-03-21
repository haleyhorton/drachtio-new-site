# Srf
Srf stands for signaling resource framework. You create an instance of Srf to create and manage SIP dialogs and messages on a drachtio server. This class provides the high-level API through which you will gain access to other classes such as [Dialogs](./), [Requests](./), and [Responses](./). 

## Constructor 

### Srf ([Tags])
Create an Srf instance. 

| Parameters      | Type         | Description
|-----------------|--------------| -----------|
| tags (optional)   | Array/String | An array of tags or a single tag to assign to this application. Tags can be used with inbound connections to a drachtio server as part of the routing decision to determine which application should be used to handle an incoming request.      

### Examples

> Create an Srf instance and connect to a drachtio server 
```js
const Srf = require('drachtio-srf');
  const srf = new Srf() ;

  srf.connect({
    host: '192.168.32.5',
    port: 9022,
    secret: 'cymru'
  }) ;
```

> Create an Srf instance and listen for outbound connections
```js
const Srf = require('drachtio-srf');
  const srf = new Srf() ;

  srf.listen({
    port: 3000,
    secret: 'cymru'
  }) ;
```

> Create an Srf instance specifying a tag for routing requests
```js
const Srf = require('drachtio-srf');
  const srf = new Srf('conference-app') ;

  srf.connect({
    host: '192.168.32.5',
    port: 9022,
    secret: 'cymru'
  }) ;
```
## Methods 

### Conect (opts)
Make an inbound connection to a drachtio server. Note that as of drachtio-srf 4.4.0 and drachtio server 0.8.0 TLS connections are supported. 

### Returns: 
 Reference to the Srf instance. 

| Parameters          | Type   | Description 
| --------------------|--------|-----------|
| opts                | Object | Configuration options
| opts.host           | string | IP address or DNS name of server to connect to
| opts.port (optional)| number | port to connect to. Default: 9022 
| opts.secret         | string | shared secret 
| opts.tls            | Object | options for establishing a TLS connection. [See here for a full list of options](./) 

### createUAS (req,res,opts, [callback])
Create a SIP dialog, acting as a UAS dialog, acting as a UAS (user agent server); i.e. respond to an incoming SIP INVITE with a 200 OK (or to a SUBSCRIBE request with a 202 Accepted). 

| Parameters                | Type     | Description
|---------------------------|----------|------------
| req                       | Request  | The incoming sip request object. 
| res                       | Response | The sip response object. 
| opts                      | Object   | Configuration options. 
| opts.localSdp (optional)  | string   | The local session description protocol to include in the SIP response. 
| opts.headers (optional)   | Object   | SIP headers to include on the SIP response to the INVITE. 
| callback (optional)       | function | If provided, callback with signature (err, dialog)

#### Returns 
a Promise if no callback is provided, otherwise a reference to the Srf instance. 

### Examples 

> Returning a Promise 
```js 
srf.invite((req, res) => {
    let mySdp; // populated somehow with SDP we want to answer in 200 OK
    srf.createUas(req, res, {localSdp: mySdp})
      .then((uas) => {
        console.log(`dialog established, remote uri is ${uas.remote.uri}`);
        uas.on('destroy', () => {
          console.log('caller hung up');
        });
        return;
      })
      .catch((err) => {
        console.log(`Error establishing dialog: ${err}`);
      });
  });
``` 

> Using a callback and populating custom headers 
```js
srf.invite((req, res) => {
    let mySdp; // populated somehow with SDP we want to answer in 200 OK
    srf.createUas(req, res, {
      localSdp: mySdp,
      headers: {
        'User-Agent': 'drachtio/iechyd-da',
        'X-Linked-UUID': '1e2587c'
      }
    }, (err, uas) => {
      if (err) return console.log(`Error creating dialog: ${err}`);
      console.log(`dialog established, local tag is ${uas.sip.localTag}`);

      uas.on('destroy', () => console.log('caller hung up'));
    });
  });
``` 
### createUAC (uri, opts, [progressCallbacks], [callback])
Create a SIP dialog, acting as a UAC (user agent client). 

| Parameters              | Type   | Description 
|-------------------------|--------|------------
| uri                     | string | request uri to send to.
| opts                    | Object | Configuration options. 
| opts. methods (optional)| string | methods of request. (Defult: INVITE) 
| opts.localSdp (optional)| string | the local session description protocol to include the SIP INVITE request 
| opts.proxy (optional)   | string | send the request through an outbound proxy, specified as full sip uri or address [:port]
| opts.headers (optional) | Object | SIP headers to include on the SIP request. 
| opts.auth (optional)    | Object I Function | Object containing sip credentials to use if challenged, or a function that returns the same. If a function is provided, it will be called with (req, res, callback) - the request that was sent as well as the response received; the function must invoke the callback with signature (err, username, password)
| opts.auth.username (optional) | string | sip username 
| opts.auth.password (optional) |string | sip password 
| progressCallbacks.cbRequest (optional) | function | callback that provides request sent over the wire, with signature (err, req)
| progressCallbacks.cbProvisional (optional) | function | callback that provides a provisional response with signature (provisionalRes)
| callback (optional) | function | If provided, callback with signature (err, dialog). 

#### Returns: 
a promise that resolves to {uac, uas} if no callback is provided, otherwise a reference to the Srf instance.

### Examples 
> simple B2BUA 
```js 
const Srf = require('drachtio-srf');
  const srf = new Srf();

  srf.connect({..});

  srf.invite((req, res) => {
    srf.createB2BUA(req, res, 'sip:1234@10.10.100.1', {localSdpB: req.body})
      .then(({uas, uac}) => {
        console.log('call connected');

        // when one side terminates, hang up the other
        uas.on('destroy', () => { uac.destroy(); });
        uac.on('destroy', () => { uas.destroy(); });
        return;
      })
      .catch((err) => {
        console.log(`call failed to connect: ${err}`);
      });
  });
``` 

> use opts.passFailure to attempt a fallback URI on failure 
```js 
const Srf = require('drachtio-srf');
  const srf = new Srf();

  srf.connect({..});

  function endCall(dlg1, dlg2) {
    dlg1.on('destroy', () => dlg2.destroy());
    dlg2.on('destroy', () => dlg1.destroy());
  }
  srf.invite((req, res) => {
    srf.createB2BUA(req, res, 'sip:1234@10.10.100.1', {localSdpB: req.body, passFailure: false})
      .catch((err) => {
        // try backup if we got a sip non-success response and the caller did not hang up
        if (err instanceof Srf.SipError && err.status !== 487) {
          console.log(`failed connecting to primary, will try backup: ${err}`);
          return {};
        }
        throw err;
      })
      .then(({uas, uac}) => {
        if (!uas) return srf.createB2BUA(req, res, 'sip:1234@10.10.100.2', {localSdpB: req.body});
        return {uas, uac};
      })
      .then(({uas, uac}) => {
        return endCall({uas, uac});
      })
      .catch((err) => {
        console.log(`failed connecting to backup uri: ${err}`);
      });
  });
``` 
> B2BUA with media proxy using rtpengine 
```js 
const Srf = require('drachtio-srf');
  const srf = new Srf();
  const config = require('config');
  const Client = require('rtpengine-client').Client;
  const rtpengine = new Client();

  srf.connect(config.get('drachtio'));

  // clean up and free rtpengine resources when either side hangs up
  function endCall(dlg1, dlg2, details) {
    [dlg1, dlg2].each((dlg) => dlg.on('destroy', () => (dlg === dlg1 ? dlg2 : dlg1).destroy()));
    rtpengine.delete(config.get('rtpengine'), details);
  }

  // function returning a Promise that resolves with the SDP to offer A leg in 18x/200 answer
  function getSdpA(details, remoteSdp, res) {
    return rtpengine.answer(config.get('rtpengine'), Object.assign(details, {
      'sdp': remoteSdp,
      'to-tag': res.getParsedHeader('To').params.tag
    }))
      .then((response) => {
        if (response.result !== 'ok') throw new Error(`Error calling answer: ${response['error-reason']}`);
        return response.sdp;
      });
  }

  // handle incoming invite
  srf.invite((req, res) => {
    const from = req.getParsedHeader('From');
    const details = {'call-id': req.get('Call-Id'), 'from-tag': from.params.tag};

    rtpengine.offer(config.get('rtpengine'), Object.assign(details, {'sdp': req.body}))
      .then((rtpResponse) => {
        if (rtpResponse && rtpResponse.result === 'ok') return rtpResponse.sdp;
        throw new Error('rtpengine failure');
      })
      .then((sdpB) => {
        return srf.createB2BUA(req, res, config.get('uri'), {
          localSdpB: sdpB,
          localSdpA: getSdpA.bind(null, details)
        });
      })
      .then(({uas, uac}) => {
        console.log('call connected with media proxy');
        return endCall(uas, uac, details);
      })
      .catch((err) => {
        console.log(`Error proxying call with media: ${err}`);
      });
  });
``` 

### endSession (msg)
Release outbound connection from the server. 
> Note: this is intended to be used with outbound connections; the method is a no-op if the underlying connection is an inbound connection 

| Parameters | Type | Description 
|------------|------|-----------
| msg        | [Request](./) I [Response](./) | A SIP request or response message recieved through a method event handler 

#### Returns: 
undefined. 

### listen (opts)
Listen for outbound connections from a drachtio server. Note that as of drachtio-srf 4.4.0 and drachtio server 0.8.0 TLS connections are supported. 

#### Returns: 
Reference to the Srf instance. 

| Parameters | Type | Description 
|------------|------|-------------
| opts | Object | Configuration options 
| opts.host (optional) | string | IP address or DNS name of server to connect to. Default: 0.0.0.0 
| opts.port | number | tcp port to listen on 
| opts.secret| string | shared secret 
|opts.tls | object | options for establishing a TLS connection. [See here for a full list of options](./)

### proxyRequest (req, [destination], [opts], [callback]) 
Proxy an incoming request. 

| Parameters | Type | Description 
| -----------|------|-------------
| req | [Request](./) | Drachtio request object representing an incoming SIP request. 
| destination (optional) | Array I string | An IP address[:port], or list of same, to proxy the request to. 
| opts (optional) | Object | Configuration options. 
| opts.forking (optional) | string | When multiple destinations are provided, this option governs whether they are attemped sequentially or in parallel. Valid values are 'sequential' or 'parallel'. Default: sequential. 
| opts.remainInDialog (optional) | string | If true, add Record-Route header and remain in the SIP dialog (i.e. receiving further SIP messaging for the dialog, including the terminating BYE request). Alias: 'recordRoute'. Default: false.
| opts.proxy (optional) | string | Send the request through an outbound proxy, specified as full sip rui or address[:port]
| opts.provisionalTimeout (optional) | string | Timeout after which to attempt the next destination if no 100 Trying response has been received. Examples of valid syntax for this property are '1500ms,' or '2s'. 
| opts.finalTimeout (optional) | string | Timeout, in miliseconds, after which to cancel the current request and attempt the next destination if no final response has been received. Syntax is the same as for the provisionalTimeout property. 
| opts.followRedirects (optional) | boolean | If true, handle 3XX redirect responses by generating a new request as per the Contact header; otherwise, proxy the 3XX response back upstream without generating a new response. Default: false. 
| callback (optional) | function | If provided, callback with signature (err, results), where 'results' is a JSON object describing the individual sip call attempts and results. 

#### Returns: 
A Promise if no callback is provided, otherwise a reference to the Srf instance.

### Examples 
> simple proxy 
```js
const Srf = require('drachtio-srf');
  const srf = new Srf();
  
  srf.invite((req, res) => {
    srf.proxyRequest(req, ['sip.example1.com', 'sip.example2.com'], {
      recordRoute: true,
      followRedirects: true,
      provisionalTimeout: '2s'
    }).then((results) => {
      console.log(JSON.stringify(result)); 
      // {finalStatus: 200, finalResponse:{..}, responses: [..]}
    });
  });
``` 

### request (uri, opts, [callback])
Send a SIP request message. 

| Parameters | Type | Description 
|------------|------|-----------
| uri | string | sip uri to send the request to. 
| opts | Object | Configuration options. 
| opts.method | string | SIP method to send. 
| opts.body (optional) | string | The body of the request. 
| opts.headers (optional) | Object | SIP headers to include on the SIP request
| opts.auth (optional) | Object | Used for digest authentication
| opts.auth.username (optional) | string | Username to provide if challenged.
| opts.auth.password (optional) | string | Password to provide if challenged. 
| callback (optional) | function | If provided, callback with signature (err, req), where 'req' represents the [SIP request](./) sent over the wire. 

#### Returns:
Undefined. 

### Examples:
> Send an OPTIONS ping 
```js 
const Srf = require('drachtio-srf');
  const srf = new Srf();

  srf.connect({host: '127.0.0.1', port: 9022, secret: 'cymru'});

  srf.on('connect', (err, hp) => {
    if (err) return console.log(`Error connecting: ${err}`);
    console.log(`connected to server listening on ${hp}`);

    setInterval(optionsPing, 10000);
  });

  function optionsPing() {
    srf.request('sip:tighthead.drachtio.org', {
      method: 'OPTIONS',
      headers: {
        'Subject': 'OPTIONS Ping'
      }
    }, (err, req) => {
      if (err) return console.log(`Error sending OPTIONS: ${err}`);
      req.on('response', (res) => {
        console.log(`Response to OPTIONS ping: ${res.status}`);
      });
    });
  }
``` 

### use ([method], handler)
Installs sip middleware, optionally invoked only for a specific method type.

| Parameters | Type | Description 
|------------|------|-----------
| method | string | A method for which the middleware is installed. Default: all methods. 
| handler | function | A SIP middleware function with signature (req, res, next).

#### Returns: 
Undefined 

### Examples 
> Install middleware to authenticate incoming requests 
```js 
const Srf = require('drachtio-srf');
  const srf = new Srf() ;
  const digestAuth = require('drachtio-mw-digest-auth');
  const regParser = require('drachtio-mw-registration-parser');

  srf.connect({...}) ;
  
  const challenge = digestAuth({
    realm: 'sip.drachtio.org',
    passwordLookup: function(username, realm, callback) {
      // ..lookup password for username in realm
      return callback(null, password) ;
    }
  }) ;

  srf.use('register', challenge) ;
  srf.use('register', regParser) ;
  srf.register((req, res) => {
    // if we reach here we have an authenticated request
    // and 'registration' property added to 'req'

    res.send(200, {
      headers: {
        'Expires': req.registration.expires
      }
    });
  });
``` 

## Events 

### "cdr:attempt" (source, time, msg)
A cdr:attempt event is emitted by an Srf instance when a call attempt has been received (inbound) or initiated (outbound). 

| Parameters | Type | Description 
|------------|------| ---------
| source | string | 'Network'l'application,' depending on whether the INVITE is inbound (received), or outbound (sent), respectively. 
| time | string | The time (UTC) recorded by the SIP stack corresponding to the attempt. 
| msg | Object | The actual message sent or received. 

### "cdr:stop" (source, time, reason, msg)
A cdr:stop event is emitted by an Srf instance when a connected call has ended. This could be either a connected call that hangs up, or an attempt taht rejected with a final non-success response. 

| Parameters | Type | Description 
| -----------|------|------------
| source | string | 'Network'l'application,' depending on whether the INVITE is inbound (receieved), or outbound (sent), respectively. 
| time | string | The time (UTC) recorded by the SIP stack corresponding to the attempt. 
| reason | string | The reason the call was ended. 
| msg | Object | The actual message was sent or received. 

### "connect" (err, hostport)
A Connect event is emitted by an Srf instance when a connect method completes with either success or failure. 

| Parameters | Type | Description 
|------------|------|------------
| err | Error | Error encountered when attempting to authorize after connecting. 
| hostport | Array | An Array of SIP endpoints that the connected drachtio server is listening on for incoming SIP messages. The format of each endpoint is protcocol/address:port, e.g. udp/127.0.0.1:5060

### "connecting" 
A connecting event is emitted by an Srf instance when it is attempting to reconnect to a server. 

### "error" (err)
An error event is emitted by an Srf instance when an inbound connection is lost. Note: the srf instance will try to automatically reconnect if (and only if) the app has established a listener for the 'error' event. 

| Parameters | Type | Description 
|------------|------|------------
| err | Error | Specific error information. 

## Properties 

### Srf.parseUri 
Static property returning a function that parses a SIP uri into components. 

#### Returns: 
Function - a function that parses a SIP uri and returns an object. 

### Examples: 
> Parse a uri 
```js 
 const Srf = require('drachtio-srf');
  const srf = new Srf();
  const config = require('config');
  const parseUri = Srf.parseUri;

  srf.connect(config.get('drachtio'));

  srf.invite((req, res) => {
    //INVITE sip:5083084809@127.0.0.1:5060;tgrp=NYC-2 SIP/2.0

    console.log(JSON.stringify(parseUri(req.uri)));
    // {
    //   "family": "ipv4",
    //   "schema": "sip",
    //   "user": "5083084809",
    //   "host": "127.0.0.1",
    //   "port": 5060,
    //    "params": {
    //      "tgrp": "NYC-2"
    //    },
    //    "headers": {}
    // }
    //
    res.send(480);
  });
``` 

### Srf.SipError 
Static property returning a SipError class. A SipError has 'status' and 'reason' properties corresponding to the sip non-success result. 

#### Returns: 
SipError - a SipError object

# Dialog 
A SIP Dialog represents a session between two SIP endpoints. You do not create a Dialog explicitly (it has no public constructor); rather, calling [srf.createUAS](./_), [srf.createUAC](./), or [srf.createB2BUA](./) will return you to a Dialog upon success. 

## Methods 

### destroy (opts,[callback])
Destroy the sip dialog by generating a BYE request (in the case of INVITE dialog), or NOTIFY (in the case of SUBSCRIBE). 

| Parameters | Type | Description 
|------------|------|-------------
| opts (optional) | Object | Configuration options. 
| opts.headers (optional) | Object | SIP headers to add to the outgoing BYE or NOTIFY. 
| callback (optional) | function | If provided, callback with signature (err, msg) where msg provides the BYE or NOTIFY message that was sent over the wire. 

#### Returns: 
A Promise that resolves with the SIP request sent if no callback is provided, otherwise a reference to the Dialog instance. 

### modify (sdp, [callback])
Modify a SIP session by changing attributes of the media connection. 

| Parameters | Type | Description 
|------------|------|-----------
| sdp | string | 'Hold,' 'unhold,' or a session description protocol. 
| callback | function | If provided, callback with signature (err) when operation has completed. 

#### Returns: 
A Promise that resolves when the operation is complete if no callback is provided, otherwise a reference to the Dialog instance. 

### request (opts, [callback])
Send a SIP request within a dialog. 

| Parameters | Type | Description 
|------------|------|------------
| opts | Object | Configuration options. 
| opts.method | string | SIP method to use for the request. 
| opts.headers (optional) | Object | SIP headers to apply on the request. 
| opts.body (optional) | string | Body of the SIP request. 
| callback (optional) | function | If provided, callback with signature (err,req) when request has been sent. 

#### Returns: 
A Promise that resolves with the request that has been sent if no callback is provided, otherwise a reference to the Dialog instance.

## Events 

### "destroy" (msg, reason)
A destroy event is emitted when the Dialog has been torn down from the far end. 

| Parameters | Type | Description 
|------------|------|------------
| msg | [Request](./) | The SIP request that tore down the Dialog. 
| reason (optional) | string | A reason the dialog was destroyed (this is populated only for error cases; e.g. an ACK timeout).

### "info" (req,res)
An info event is emitted when a SIP INFO request is received within the Dialog. When an application adds a handler for this event it must generate the SIP response by calling res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|------------|------|-----------
| req | [Request](./) | A SIP request. 
| res | [Response](./) | A SIP response. 

### "message" (req,res)
A message event is emitted when a SIP MESSAGE request is received within the Dialog. When an application adds a handler for this event it must generate the SIP response by callin res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|-----------|-----|------------- 
| req | [Request](./) | A SIP request. 
| res | [Response](./) | A SIP response. 

### "modify" (req,res)
A modify event is triggered when the far end modifies the session by sending a re-INVITE. When an application adds a handler for this event it must generate the SIP response by calling res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|------------|------|---------
| req | [Request](./) | A SIP request 
| res | [Response](./) | A SIP response 

### "options" (req,res) 
An options event is emitted when a SIP OPTIONS request is received within the Dialog. When an application adds a handler for this event it must generate the SIP response by calling res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|------------|------|---------
| req | [Request](./) | A SIP request 
| res | [Response](./) | A SIP response 

### "refer" (req,res)
A refer event is emitted when a SIP REFER request is received within the Dialog. When an application adds a handler for this event it must generate the SIP response by calling res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|------------|------|---------
| req | [Request](./) | A SIP request 
| res | [Response](./) | A SIP response 

### "refresh" (req)
A refresh event is emitted when a SIP refreshing re-INVITE is received within the Dialog. There is no need for the application to respond to this event; this is purely a notification. 

| Parameters | Type | Description 
|------------|------|---------
| req | [Request](./) | A SIP request 

### "update" (req,res)
An update event is emitted when a SIP UPDATE request is received within the Dialog. When an application adds a handler for this event it must generate the SIP response by calling res.send on the provided drachtio response object. When no handler is found for this event a 200 OK will be automatically generated. 

| Parameters | Type | Description 
|------------|------|---------
| req | [Request](./) | A SIP request 
| res | [Response](./) | A SIP response

## Properties 

### dialog Type 
'INVITE' or 'SUBSCRIBE' depending on the Dialog type. 

### id 
String that provides a unique internal id for the Dialog. 

### local 
Object containing information about the local side of the Dialog. 

| Parameters | Type | Description 
|------------|------|---------
| uri | string | Local sip uri. 
| sdp | string | Local sdp. 
| contact | string | Local contact header. 

### remote 
Object containing information about the remote side of the Dialog. 

| Parameters | Type | Description
|------------|------|--------- 
| uri | string | Remote sip uri. 
| sdp | string | Remote sdp. 
| contact | string | Remote contact header.

### sip 
Object containing information about the SIP details relating to the Dialog. 

| Parameters | Type | Description
|------------|------|---------
| callId | string | SIP Call-ID for the Dialog. 
| localTag | string | Tag generated by local side of the Dialog. 
| remoteTag | string | Tag generated by remote side of the Dialog. 

### subscribeEvent 
String that indicates the Event subscribed for in a SUBSCRIBE Dialog. 

# SipMessage 
This is a mix-in class containing common methods and properties for [Request](./) and [Response](./) objects. You generally do not work directly with SipMessage objects, rather with Request or Response objects provided via calllbacks or Promises. 

## Methods 

### get(hdr)
Returns a header value as a string. 

| Parameters | Type | Description
|------------|------|---------
| hdr | string | Header name. 

#### Returns: 
A string containing the header value, or undefined if no such header exists. 

### Examples: 
> Getting a header value 
```js 
srf.invite((req, res) => {
    console.log(req.get('Via'));
    // SIP/2.0/UDP 172.16.0.129:5061;branch=z9hG4bK-1859-1-0;received=127.0.0.1;rport=5061
``` 

### getParsedHeader (hdr)
Returns a header value as an Object containing individual information elements.

| Parameters | Type | Description
|------------|------|---------
| hdr | string | Header name.

#### Returns: 
An object containing the parsed header elements, or undefined if no such header exists. 

### Examples: 
> Getting a parsed header 
```js 
srf.invite((req, res) => {
    console.log(JSON.stringify(req.getParsedHeader('Via')));
    /*
    [{
      "version": "2.0",
      "protocol": "UDP",
      "host": "172.16.0.129",
      "port": 5061,
      "params": {
        "branch": "z9hG4bK-2070-1-0",
        "received": "127.0.0.1",
        "rport": "5061"
      }
    }]
    */
``` 

### has(hdr)
Returns a boolean indicating if a header is present. 

| Parameters | Type | Description
|------------|------|---------
| hdr | string | Header name.

#### Returns: 
Boolean indicating presence of header in the message. 

## Properties 

### body 
String providing the body of the message, if any.

### calledNumber 
String providing the phone number in the request uri (request only).

### callingNumber 
String providing the calling phone number found in the P-Assered-Identity or From header (request only).

### headers 
Object containing all of the SIP headers for this message. 

### method 
String providing the method (request only). 

### payload 
Array containing the body of the message organized into parts. The payload property is similar to the body property, in that both provide access to the information carried in the body of the request. However, whereas req.body provides the raw uft-8 string as carried on the wire, req.payload structures the information into parts. This is primarily useful when dealing with multipart bodies. 

### Examples 
> Using req.payload to work with multipart content 
```js 
// e.g., siprec INVITE has multipart content
  srf.invite((req, res) => {
    if (req.payload.length > 1) {
      console.log(`we have multipart content: ${JSON.stringify(req.payload)}`);
    }
    /*
    we have multipart content: 
    [{
      "type": "application/sdp",
      "content": "v=0\r\no=Sonus_UAC 328001 655768 .."
    }, {
      "type": "application/rs-metadata+xml",
      "content": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\r\n<tns:recording ..."
    }]                  
    */
``` 

### protocol 
String indicating the transport protocol used (e.g. "udp," "tcp").

### reason 
String that is the SIP reason code (response only). 

### source 
String that indicates source of message: "network" or "application." 

### source_address 
String that indicates the sending IP address. 

### Example 
> Accessing IP address and port of sender 
```js 
srf.invite((req, res) => {
    console.log(`received INVITE from ${req.source_address}:${req.source_port}`);
    .. etc
  });
``` 

### source_port 
String that indicates the sending IP port. 

### stackTime 
String that indicates the time the SIP stack handled the message. 

### stackDialogId 
String that indicates the unique id assigned to the Dialog by the SIP stack (INVITE or SUBSCRIBE only). 

### stackTxnId 
String that indicates the unique id assigned to the transaction by the SIP stack.

### status 
Number that is the SIP status code (response only). 

### type 
'Request' or 'response.' 

### uri 
String containing the request-uri (request only). 

# Request 
Represents a SIP request that is either received by or sent from an application. Note that [SipMessage](./) is mixed into this class, so all of those methods and properties are available as well. 

## Methods 

### cancel ([callback])
Cancels a request that was sent by the application. 

| Parameters | Type | Description 
|------------|------|------------
| callback (optional) | function | Callback that is invoked with signature (err, cancel), where 'cancel' is the SIP CANCEL [request](./) that was sent over the wire. 

#### Returns: 
Undefined. 

### proxy ([opts], [callback])
Proxy an incoming request. Note: it is generally preffered to call [srf.proxyRequest](./). 

| Parameters | Type | Description 
|-----------|-----|------------
| opts | Object | Proxy options, as defined in [srf.proxyRequest](./)
| callback (optional) | function | Callback that is involved with the results of the proxy operation. 

#### Returns: 
A Promise that resolves with the proxy results if no callback is provided, otherwise a reference to the Request. 

## Properties 

### isNewInvite 
Boolean indicating if this is an INVITE that is not part of an existing Dialog. 

## Events 

### "cancel" 
A cancel event is emitted when an incoming Request has been canceled by the sender. 

### "response (msg)" 
A response event is emitted when a response has been received for the request which has been sent by the application. 

| Parameters | Type | Description 
|-----------|-----|------------
| msg | [Response](./) | A SIP response message that was received in response to a request by the app. 

# Response 
Represents a SIP response that is either received by or sent from an application. Note that [SipMessage](./) is mixed into this class, so all of those methods and properties are available as well. 

## Methods 

### send (status, [reason], [opts], [callback], [fnAck])
Sends a SIP response. 

| Parameters | Type | Description 
|-----------|-----|------------
| status | number | SIP status code to send. 
| reason (optional) | string | SIP reason code to send. Default: the predefinied reason code associated with the provided status. 
| opts (optional) | Object | Configuration options. 
| opts.headers (optional) | Object | SIP headers to send with response. 
| opts.body (optional) | string | Body send with response. 
| callback (optional) | function | Callback invoked with signature (err, response), where 'response' is the SIP response sent over the wire. 
| fnAck (optional) | function | Function to be executed upon receipt of ACK or PRACK. 

#### Returns: 
Undefined. 

## Properties 

### finalResponseSent 
Boolean indicating if a final response has been sent (alias: 'headersSent'). 

### statusCode 
Number indicating the SIP status code that was sent or received. 