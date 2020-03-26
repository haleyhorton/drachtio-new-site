## 4.4.27

* bugfix: copyAllHeaders needs to handle case where INVITE could not be sent

## 4.4.26

* proxyRequestHeaders and proxyResponseHeaders now can take keyword 'all' to copy SIP headers from A to B leg without specifying them individually.  Can also then selectively remove some; e.g. `proxyRequestHeaders: ['all', '-Authorization']`.  Note that headers such as To, From, Via, Contact, Route, Record-Route are never copied.

## 4.4.24 

* when calling fnAck provided in Response#send callback, pass the ACK/PRACK message as a parameter (partial fix for [#70](https://github.com/davehorton/drachtio-srf/issues/70))

## 4.4.23

* UAC: when sending request with credentials after challenge, make sure two From headers are not included (drachtio server issue 106)
* allow auth on in-dialog requests [#67](https://github.com/davehorton/drachtio-srf/issues/67)

## 4.4.22

* Merge pull request #63 from Telzio/bugfix-isNewDialog
* for new incoming requests outside of a dialog, req.receiveOn contains the local sip endpoint that received the incoming request [65](https://github.com/davehorton/drachtio-srf/issues/65)

## 4.4.20

* race condition fix: b2bua, must wait for ACK from A before releasing queue of pending requests from B [#62](https://github.com/davehorton/drachtio-srf/issues/)

## 4.4.19

* Fixed bug where request.js calls non-existent method
* Merge pull request #61 from Telzio/feature-hold-unhold-events
* Only apply refresh/hold/unhold logic if these events are subscribed

## 4.4.18

* [#55](https://github.com/davehorton/drachtio-srf/issues/55) allow customization of From header; also remove unused api logging feature

## 4.4.15

* bugfix: when b2b fails to send to B due to stack error or warning, no response was propgated to A

## 4.4.14

* [#15](https://github.com/davehorton/drachtio-srf/issues/51) - added attributes to `req` attribute that describe the drachtio server the request is coming from:

- `req.server.address`: the IP address the drachtio server sending the request.  Note: this is the IP address the server is bound to for the tcp/tls socket connection to the application, which may be different than the IP address(es) that the server is listening on for SIP traffic.
- `req.server.hostport`: a list of SIP addresses and associated protocols that the drachtio server is listening on.  This is the same information that is returned in the callback to the [Srf#connect](https://drachtio.org/api#srf-event-connect) event

## 4.4.13

* [#48](https://github.com/davehorton/drachtio-srf/issues/48) fix memory leak in outbound scenario

## 4.4.12 

* [#47](https://github.com/davehorton/drachtio-srf/issues/47) when using outbound connections CANCEL was sometimes sent to wrong server in B2B scenario

## 4.4.11 

* Dialog#modify now resolves to remote SDP received from reinvite

## 4.4.10 

* allow app to set explicitly tag on To header when responding
* bugfix: Honour original uri transport in challenge response fixes - [issue 45](https://github.com/davehorton/drachtio-srf/issues/45)

## 4.4.9 

* bugfix: race condition with immediate request within newly-formed dialog - [issue 43](https://github.com/davehorton/drachtio-srf/issues/43)

## 4.4.8

* Added support for sending periodic 'ping' requests to the server.  These are mainly useful to keep connections alive and active if they traverse network firewalls that may terminate tcp connections periodically if traffic has not been exchanged between the endpoints.  This feature is disabled by default, but can be turned on by setting `opts.enablePing` when calling Srf#connect or Srf#listen, e.g.

<pre style="margin-left: 5em; color:#D91C5C">
srf.connect({
  host: '127.0.0.1',
  port: 9022,
  secret: 'cymru',
  enablePing: true
});
</pre>

The default ping interval is 15 seconds.  You can change this by setting `opts.pingInterval` to a value of milliseconds (must be >= 5000 and <= 300000); for instance, to set the ping interval to 45 seconds:

<pre style="margin-left: 5em; color:#D91C5C">
srf.connect({
  host: '127.0.0.1',
  port: 9022,
  secret: 'cymru',
  enablePing: true,
  pingInterval: 45000
});
</pre>

The server responds with a 'pong' message (note that you do not need to do anything to handle this; it is not exposed to the application level).

This feature requires a drachtio server version of v0.8.2 or above; however, it is safe to call it against earlier versions (there just won't be any ping messages sent).

## 4.4.7 

* add Srf#findDialogById and Srf#findDialogByCallIDAndFromTag, the latter is particularly useful in handling attended transfer scenarios per [issue #34](https://github.com/davehorton/drachtio-srf/issues/34)
* migrated by istanbul to nyc for code coverage

## 4.4.6

* make metadata available with destroy and ack events
* bugfix: parse sip scheme properly ([pull request #33](https://github.com/davehorton/drachtio-srf/pull/33))

## 4.4.5

* pass request request metadata in dialog events; e.g. so when handling a BYE metadata such as the stack time of the message is available.

## 4.4.4

* bugfix: opts.localSdpA in Srf#createB2BUA should allow for a function returning a string, as documented.  (Other options are for opts.localSdpA simply to be a string, or a function returning a Promise that resolves to a string).

## 4.4.3

* update to latest drachtio-sip with bugfix for parsing multipart content-type header (fixes bug identified when handling SIPREC invite from Oracle/Acme)
* update to latest drachtio-sip to remove security warning

## 4.4.2

* add algorithm=MD when constructing Authorization header (this is optional per spec, but some require it)

## 4.4.1

* When handling a Digest challenge as a UAC after sending the initial request to a DNS name, send the follow-up credentialled request to the responding server ip address, rather than doing another DNS lookup.
* Export Request and SipMessage classes (mainly needed to construct test scenarios).

## 4.4.0

* Added optional support for TLS encryption of messages between the drachtio server and drachtio-srf applications.  Requires drachtio-server@0.8.0-rc1 or above.

## 4.3.7 

* Fixed bug parsing messages from drachtio server that was causing intermittent "invalid message from server, did not start with.." errors
* when receiving a CANCEL that we cant relate to a pending INVITE, silently discard instead of returning 404

## 4.3.6 

* Fixed race condition when app CANCELs a uac INVITE in the midst of handling a 401/407 challenge response.

## 4.3.5

* Fixed bug in Dialog#handle where an incoming NOTIFY request with a Subscription-State header was assumed to be for a SUBCRIBE dialog - this is not the case for a NOTIFY after a REFER.
* Fix bug in Digest authentication: nc and algorithm values should not be in quotes in Authorization (or Proxy-Authorization) header.

## 4.3.4

* Added `opts.responseHeaders` to [Srf#createB2BUA](/api#srf-create-b2bua) to provide SIP headers to include on responses to the A party.  The value may be either an object containing the SIP headers to apply on all responses to the A party (provisional as well as final), or a function returning an object that contains the SIP headers to apply.  If a function is provided, it will be called with the signature `(uacRes, headers)`, where `uacRes` is the response received from the B party, and `headers` are the SIP headers that have already been set on the response message to the A party (e.g., through the `opts.proxyResponse` option).

## 4.3.3

* Fixed uac scenario (`srf.createUAC` or `srf.createB2BUA`) where `opts.auth` is provided and 401/407 challenge is handled.  The stack transaction for the initial request was being incorrectly used to send the CANCEL or ACK for the subsequent request (the one carrying credentials).

## 4.3.2

* Fixed regression bug in recent change to [Srf#request](/api#srf-request).

## 4.3.1 

* Fix bug in [Dialog#request](/api#dialog-request) where Promise was not being resolved properly with the received response to the outbound request.
* [Srf#proxyRequest](/api#srf-proxy-request) and [Srf#request](/api#srf-request) now return Promises when no callback provided, for consistency with other methods

## 4.3.0 


* Added support for tagged inbound connections.
* Bundled functionality that was originally in the drachtio module; package.json now only requires drachtio-srf (i.e. do not include drachtio as a dependency).
