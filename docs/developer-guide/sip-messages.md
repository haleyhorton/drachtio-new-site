# SIP Messages

In the examples above, we've seen the callback signature `(req, res)` through which we are passed objects representing a SIP request and an associated response.  These objects are event emitters and have some useful properties and methods attached.  Since we will be interacting with these objects a lot when writing applications, let's review them now.

## Properties, methods and events

The following properties are available on both `req` and `res` objects:

* `type`: 'request' or 'response'
* `body`: the SIP message body, if any
* `payload`: an array of content, useful mainly if the message included multipart content.  Each object in the payload array has a `type` and `content` property, containing the Content-Type header and the associated content, respectively 
* `source`: 'network' or 'application'; the sender of the message
* `source_address`: the IP address of the sender
* `source_port`: the source port of the sender
* `protocol`: the transport protocol being used (e.g., 'udp', 'tcp')
* `stackTime`: the time the message was sent or received by the drachtio server sip stack
* `calledNumber`: the phone number (if any) parsed from the user part of the request uri
* `callingNumber`: the phone number (if any) of the calling party, parsed from the P-Asserted-Identity header if it exists, otherwise from the From header.
* `raw`: a string containing the full, unparsed SIP message

The  following methods are available on both `req` and `res` objects as well:

* `has(name)`: returns true if the message includes the specified header
* `get(name)`: returns the value of a specified SIP header
* `set(name, value)`: sets the value of a specified SIP header
* `getParsedHeader(name)`: returns an object that represents the specified SIP header parsed into components
### Request-specific properties, methods and events

The `req` object additionally has the following properties:

* `method`: the SIP method of the request

the following methods:

* `isNewInvite()`: returns true if the request is a new INVITE (vs a re-INVITE, or a non-INVITE request)
* `cancel(callback)`: cancels an INVITE request that was sent by the application
* `proxy(opts, callback)`: proxies an incoming request.  While this method is available, the preferred usage is to call `srf.proxyRequest()` instead.

and emits the following events:

* `cancel`: this event is emitted for an incoming INVITE request, when a CANCEL for that INVITE is subsequently received from the sender.
* `response`: when an application sends a SIP request, an application can listen for the 'response' event to obtain the matching SIP response that is received.
### Response-specific properties, methods and events

The `res` object additionally has the following properties:

* `status`: the SIP response status, as an integer (alias: `statusCode`)
* `reason`: the SIP reason (e.g. 'Busy Here')
* `finalResponseSent`: true if the response message has been sent (alias: `headersSent`)

and the following methods:

* `send(status, reason, opts, callback)`: we have already seen this method used to send a response.  Only the status parameter is required. The callback, if provided, will be invoked with the signature `(err, msg)` where the `msg` parameter will contain a representation of the SIP response message sent out over the wire.

## Usage patterns

In the few sample code snippets we've looked at so far, we have been receiving SIP requests and then sending SIP responses in return.  

However, we can also do the reverse -- send out a SIP request and receive a response back.  In either case, we are dealing with the request and response objects described above, but different methods and events may apply.  Below some of the common patterns are covered.

#### Receiving a request and sending a response
```js
srf.options((req, res) => {
  res.send(200);
});
```
#### Sending a response with headers
```js
srf.options((req, res) => {
  res.send(200, {
    headers: {
      'Subject': 'All\'s well here'
    }
  });
});
```
#### Sending a response with callback to get the msg actually sent
```js
srf.options((req, res) => {
  res.send(200, {
    headers: {
      'Subject': 'All\'s well here'
    }
  }, (err, msg) => {
    const to = msg.getParsedHeader('To');
    console.log(`drachtio server added tag on To header: ${to.params.tag}`);
  });
});
```
#### Sending a request, and then receiving the response
```js
srf.request('sip:1234@example.com', {
  method: 'OPTIONS'
}, (err, req) => {
  // req is the SIP request that went out over the wire
  req.on('response', (res) => {
    console.log(`received ${res.status} response to our OPTIONS request`);
  });
});
```
#### Sending a request with headers and body
```js
const dtmf = 
`Signal=5
Duration=160`;

srf.request('sip:1234@example.com', {
  method: 'INFO', 
  headers: {
    'Content-Type': 'application/dtmf-relay'
  },
  body: dtmf
});
```
#### Handling the cancel of an INVITE
```js
srf.invite((req, res) => {
  let canceled = false;

  req.on('cancel', () => canceled = true);

  doLengthyDatabaseLookup()
    .then((results) => {
      
      // was the call canceled while 
      // we were doing database lookup?

      if (canceled) return;

      ..go on to process the call
  })
})
```