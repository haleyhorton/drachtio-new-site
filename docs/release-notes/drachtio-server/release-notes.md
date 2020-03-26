## 0.8.4 

* fix travis osx build
* add support for specifying number of log files to keep
* bugfix: crash sending ACK to target with dns name that needs to be resolved (110)
* bugfix - crash when RR header in 200 OK response has dns name (#110)
* add tests for handling reinvite with null sdp and invoking fnAck
* add dialogid to msgs back to app when sending a response within a dialog
* pass ip address:port new incoming request was received on to client applications
* bugfix: address to bind to for client connections was being ignored (#103)
* add support for adding headers to CANCEL request
* add support for tcp keepalive
* return 491 Request Pending if we receive a re-INVITE while still processing a previous INVITE transaction
* travis change to build node 10.x for tests
* temp revert nta_outgoing_destroy due to crash (#76)


## 0.8.2 

* simplify timerD handling, fix some memory leaks related to not destroying sofia transactions in UAC scenarios
* change dialog id naming convention to "{call-id};from-tag={from-tag}", which more correctly maps to SIP standard and simplifies transfer scenarios in applications
* [add support for detecting public ip](https://github.com/davehorton/drachtio-server/blob/98f20d7b0cd672b275e048ba759673e2d99a2fc5/entrypoint.sh#L19) when running in a Docker container in Azure cloud
* include server version in response to authenticate command (allows clients to activate version-specific features). Version info is available to client (when using [drachtio-srf@4.4.8](https://www.npmjs.com/package/drachtio-srf) or above, and connecting to a server at version 0.8.2 or above) as a parameter in the Srf#connect event, e.g.: 
```js
  srf.on('connect', (err, hp, serverVersion) => {
    console.log(serverVersion);
    // "v.0.8.2"
  });
```
* bugfix: generate proper CDRs for B2B scenarios
* add support for 'ping' request from client

## 0.8.1

* added support for [prometheuos.io monitoring](https://drachtio.org/docs/drachtio-server#monitoring-section)
* added support for configuring some settings via environment variables
* update to boost 1.69
* handle an incoming NOTIFY or request before dialog is formed (issue [#75](https://github.com/davehorton/drachtio-server/issues/75))
* changes to build properly on arm64
* update to latest sofia

## 0.8.0

* bugfix: crash when requested protocol is not enabled (issue #64)
* add support for nat detection based on appearance of nat=yes in the Record-Route or Contact header (issue #66).  Some proxies are known to insert this when the detect nat scenarios.
* fix for some compile warnings.
* when choosing a transport for outbound invite, check the the protocol of the outbound proxy (if one is configured)
* include a transport= param in the Contact header in 200 OK response to incoming invite when transport is not udp (required by pjsip)
* add support for cloud deployments in Kubernetes where the public IP address must be dynamically discovered.
* added autogen.sh deprecating bootstrap.sh
* Major changes here to remove a lot of boost code which is now included in the C++ standard.  __Most notably, gcc 4.9 or above is now required.__
* Compile with stdc++-17 if available, stdc++-11 if not.
* handle a mid-call network handoff, e.g. a client on a wifi network switches to LTE during a call - BYE should be sent to the new (on the LTE network) address.
* send ACK to failed response using same transport as response was received on.
* fix for sending to non-standard syslog UDP port, if specified in config.
* Added optional support for TLS encryption of messages between the drachtio server and drachtio-srf applications.  Requires drachtio-srf@4.4.0 or above.
* set timerG and timerH only for udp transports.
* handle SIGPIPE without exiting - can happen occasionally when a client disconnects.
* eliminate confusing / contradictory GPL license text.

## 0.7.4

* Added support for establishing uas INVITE dialogs with clients that are behind a nat.
* fix crashing bug in non-UDP transport scenarios that was introduced in 0.7.4-rc1

## 0.7.3

* Added support for tagged inbound connections; requires [drachtio-srf@4.3.0](/releases) or above.
* client-server messaging changed from ascii to utf-8 in order to support message bodies with things like emojis.
* various Dockerfile fixes.
* run-time tests added and travis-ci changes to accomodate them.
* sofia-sip has an annoying feature where it forces an outbound request to go out TCP if the packet size exceeds a specific threshold (usually 1300 bytes).  A new configuration setting was added to allow users to increase this threshold to an arbitrary value.  This value can be provided in the configuration file, or via a command-line parameter, as illustrated below.

**in drachtio.conf.xml**
<textarea rows="3" cols="40" style="border:none;">
  <sip>
    <udp-mtu>4096</udp-mtu>
  </sip>
</textarea>

**or, on command-line**
<textarea rows="1" cols="40" style="border:none;">
  drachtio --mtu 4096
</textarea>

* Some clients (e.g. Tandberg) have an annoying habit of included a Route header on an initial INVITE; i.e. a "preloaded" Route header.  When proxying such an INVITE, the server now strips that header before sending the request onwards, so as not to confuse downstream elements.
* Ensure that 'from' tag on new outbound SIP requests use the local tag on the nta leg.  There were cases when an INVITE was challenged that the subsequent INVITE w/ authorization credentials was using the old tag from the completed leg, rather than the new leg created for the second request.