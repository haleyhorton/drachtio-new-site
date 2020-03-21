# SIP Dialogs

Conceptually, a SIP dialog is defined in [RFC 3261](https://tools.ietf.org/html/rfc3261) as a relationship between two SIP endpoints that persists for a period time.  Generally speaking, we are referring most often to a multimedia call initiated by a SIP INVITE transaction during which audio and/or video is exchanged. (Later we will discuss an alternative type of dialog created by a SUBSCRIBE transaction).

Within drachtio, A SIP dialog is an object that is created to represent a multimedia session and to allow a developer to manage such sessions: to create them, modify them, and tear them down.  

The examples we have shown till now have illustrated how to manage SIP interactions at the SIP message level. However, in most cases the drachtio SIP Dialog API provides a higher-level abstraction that makes it easier for developers to manage sessions.

## User Roles in a Dialog
Some basic terminology is going to be helpful before diving into the API and some examples.

A drachtio application can:
* initiate an INVITE transaction -- in which case we say it is acting as a 'User Agent Client', or UAC,
* respond to an incoming INVITE transaction -- in which case we call it a 'User Agent Server', or UAS, and/or
* *both* receive *and* send an INVITE transaction -- in which case we call it a 'back to back User Agent', or B2BUA

By the way, these are not the only types of applications we can build with drachtio. We can also build:
* a registar -- which is an application that responds to REGISTER requests
* a presence server -- which is an application that responds to SUBSCRIBE requests
* an instant messaging server -- which responds to MESSAGE requests ([see here](https://github.com/davehorton/drachtio-simple-server) for an example of a presence and messaging server)
* a sip proxy -- which is an application that routes and forwards SIP requests

Quite frequently, though, we will find ourselves wanting to build some form of SIP User Agent application, and that is what we will cover in this section.

The dialog API consists of the methods
* [Srf#createUAS](/docs/api#Srf+createUAC), 
* [Srf#createUAC](/docs/api#Srf+createUAC), and 
* [Srf#createB2BUA](/docs/api#Srf+createB2BUA).  

Each of these methods produces a [Dialog](/docs/api/#Dialog) object when a session is successfully established, which is returned via a callback (if provided) and resolves the Promise returned by each of the above methods.

Before we review the above three methods, let's examine the [Dialog](/docs/api/#Dialog) class itself, and how to work with it.

## Dialog

The Dialog class is an event emitter, and has the following properties, methods, and events.

### properties
* `sip`: an object containing properties that identify the SIP dialog
* `sip.callId`: the SIP Call-Id associated with this dialog
* `sip.remoteTag`: the remote tag associated with the dialog
* `sip.localTag`: the local tag associated with the dialog
* `local`: an object containing properties associated with the local end of the dialog
* `local.sdp`: the local session description protocol
* `local.uri`: the local sip uri 
* `local.contact`: the local contact
* `remote`: an object containing properties associated with the remote end of the dialog
* `remote.sdp`: the remote session description protocol
* `remote.uri`: the remote sip uri 
* `remote.contact`: the local contact
* `id`: a unique identifier for the dialog with the drachtio framework
* `dialogType`: either 'INVITE' or 'SUBSCRIBE'

### methods
* [destroy(opts, callback)](/docs/api#Dialog+destroy): terminates a dialog, by sending a BYE (in the case of an INVITE dialog), or a NOTIFY with Subscription-State: terminated (in the case of a SUBSCRIBE dialog).
* [modify(sdp, callback)](/docs/api#Dialog+modify): modifies a SIP INVITE dialog by putting it on or off hold or re-INVITing to a new session description protocol.
* [request(opts, callback)](/docs/api#Dialog+request): sends a SIP request within a dialog.

### events
* [destroy(req)](/docs/api#Dialog+event_destroy): emitted when the remote end has terminated the dialog.  The `req` parameter represents the SIP request sent from the remote end to terminate the dialog.  (Note: no action is required by the application, as the drachtio server will have sent a 200 OK to the request).
* [hold(req)](/docs/api#Dialog+event_hold): emitted when the remote end has placed the call on hold.  The `req` parameter represents the INVITE on hold sent. (Note: no action is required by the application, as the drachtio server will have sent a 200 OK to the request).
* [modify(req,res)](/docs/api#Dialog+event_modify): emitted when the remote end has sent a re-INVITE with a changed session description protocol.  The application must respond, using the `res` parameter provided.
* [refresh(req)](/docs/api#Dialog+event_refresh): emitted when the remote end has sent a refreshing re-INVITE.  (Note: no action is required by the application, as the drachtio server will have sent a 200 OK to the request).
* [unhold(req)](/docs/api#Dialog+event_unhold): emitted when the remote end has taken the call off hold.  The `req` parameter represents the INVITE off hold sent. (Note: no action is required by the application, as the drachtio server will have sent a 200 OK to the request).
* [info(req, res)](/docs/api#Dialog+event_info)
* [messsage(req, res)](/docs/api#Dialog+event_message)
* [options(req, res)](/docs/api#Dialog+event_options)
* [publish(req, res)](/docs/api#Dialog+event_publish)
* [refer(req, res)](/docs/api#Dialog+event_refer)
* [subscribe(req, res)](/docs/api#Dialog+event_subscribe)
* [update(req, res)](/docs/api#Dialog+event_update): these and the above events are emitted when a SIP request of the specified type is received within a dialog.  The application must respond, using the `res` parameter provided.  If the application does not have a listener registered for the event, then the drachtio server will automatically respond with a 200 OK.

> Pro tip: while there are many operations you might want to perform on a Dialog object, the one thing you should *always* do is to listen for the 'destroy' event.  You should attach a listener for 'destroy' whenever you create a new dialog.  This will tell you when the remote side has hung up, and after this you will no longer be able to operate on the dialog.

## UAS

Whew!  With that background under our belt we can finally get to the meat of the matter -- creating and managing calls.

When we receive an incoming call and connect it to an IVR, or to a conference bridge, our application is acting as a UAS. Let's look at these scenarios first.

The one piece of information we need when acting as a UAS is the session description protocol (sdp) that we want to offer in our 200 OK.  Creating media endpoints is outside the scope of drachtio-srf, so the examples below assume that our application has obtained them through other means.
> Note: check out [drachtio-fsmrf](https://www.npmjs.com/package/drachtio-fsmrf), which is a npm module that can be used with drachtio-srf to control media resources on a [Freeswitch](https://freeswitch.com/) media server in order to provide IVR and conferencing features to drachtio applications.

```js
srf.invite((req, res) => {
  let sdp = 'some-session-description-protocol'
  srf.createUAS(req, res, {
    localSdp: sdp  
  })
    .then((dialog) => {
      console.log('successfully created UAS dialog');
      dialog.on('destroy', () => console.log('remote party hung up'));
    })
    .catch((err) => {
      console.log(`Error creating UAS dialog: ${err}`);
    }) ;
});
```
In the example above, the local sdp is provided as a string, but we can alternatively provide a function that returns a Promise which resolves to a string value representing the session description protocol.  This is useful when we have to perform some sort of asynchronous operation to obtain the sdp.
```js
function getMySdp() {
  return doSomeNetworkOperation()
    .then((results) => {
      return results.sdp;
    });
}
srf.invite((req, res) => {
  let sdp = 'some-session-description-protocol'
  srf.createUAS(req, res, {
    localSdp: getMySdp
  })
    .then((dialog) => { .. })
    .catch((err) => { .. });
});
```
Of course, we can supply SIP headers in the usual manner:
```js
srf.invite((req, res) => {
  srf.createUAS(req, res, {
    localSdp: sdp, 
    headers: {
     'User-Agent': 'drachtio/iechyd-da',
     'X-Linked-UUID': '1e2587c'
    }
  })
    .then((dialog) => { .. });
});
```
If [Srf#createUAS](/docs/api#Srf+createUAS) fails to create a dialog for some reason, an error object will be returned via either callback or the Promise.  If the failure is due to a SIP non-success status code, then a [SipError](/docs/api#siperror) will be returned.  In the UAS scenario, the only time this will happen is if the call is canceled by the caller before we answer it, in which case a '487 Request Terminated' will be the final SIP status.
```js
srf.invite((req, res) => {
  srf.createUAS(req, res, {
    localSdp: sdp, 
    headers: {
     'User-Agent': 'drachtio/iechyd-da',
     'X-Linked-UUID': '1e2587c'
    }
  })
    .then((dialog) => { .. })
    .catch((err) => {
      if (err instanceof Srf.SipError && err.status === 487) {
        console.log('call canceled by caller');
      }
    })
```

Finally, as noted above, [Srf#createUAS](/docs/api#Srf+createUAS) can be invoked with a callback as an alternative to Promises.  In most of the examples in this document we will use Promises, but an example of using a callback is presented below.

```js
srf.invite((req, res) => {
  let sdp = 'some-session-description-protocol'
  srf.createUAS(req, res, {
    localSdp: sdp  
  }, (err, dialog) => {
    if (err) console.log(`Error creating UAS dialog: ${err}`);
    else {
      console.log('successfully created UAS dialog');
      dialog.on('destroy', () => console.log('remote party hung up'));
    }
  });
});
```

## UAC

When we initiate a dialog by sending an INVITE, we are acting as a UAC.  We use the [Srf#createUAC](/docs/api#Srf+createUAC) method to accomplish this.  Just as with [Srf#createUAS](/docs/api#Srf+createUAS), either a callback approach or a Promise-based approach is supported.

In the simplest example, we can provide only the Request-URI we want to send to, and the session description protocol we are offering in the INVITE in that example:
```js
let mySdp; // populated somehow with SDP we want to offer
srf.createUac('sip:1234@10.10.100.1', {localSdp: mySdp})
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```
> Pro tip: we can specify the Request-Uri either a full sip uri, as above, or simply provide an ip address (or ip address:port, if we want to send to a non-default SIP port).

### What did drachtio server do in all this?
In the example above, we supplied only the Request-URI and body of the INVITE, so the drachtio server must have done quite a bit in terms of filling out the rest of the message and managing various aspects of the transaction.  

In fact, the drachtio server would have done all of the following to create the outgoing INVITE:
* generated a (unique) SIP Call-ID and a CSeq
* generated the appropriate Via header (pro tip: never provide a Via: header, always let drachtio generate that)
* generated the From header (with no user element in the URI), and generated a From tag
* generated the To header (according to the Request-URI we provided)
* generated a Contact header, based on the Request-URI and the address:port(s) that the server is listening on for SIP messages
* examined the body that we provided and -- seeing that it was an sdp -- created a Content-Type of 'application/sdp'
* calculated the Content-Length header (pro tip: never provide a Content-Length: header, always let drachtio generate that)

It would have then sent the INVITE, and managed any provisional and final responses.  It would have generated the final ACK request as well.  If a reliable provisional response were received, it would have responded with the required PRACK request.

Beyond this basic usage, there are several other common patterns.  Let's look at some of them.

### Receiving provisional responses

When we send out an INVITE, we may get some provisional (1XX) responses back before we get a final response.  In the example above, we did not care to do anything with these provisional responses, but if we want to receive them we can add a callback that will be invoked when we receive a provisional response.  This callback, named `cbProvisional`, along with another we will describe shortly, are provided in an optional Object parameter as shown below.

```js
srf.createUac('sip:1234@10.10.100.1', {localSdp: mySdp}, {
    cbProvisional: (res) => console.log(`got provisional response: ${res.status}`))
  })
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err.status}`);
  });
```
> Note: if the INVITE fails, a [SipError](/docs/api#siperror) object will be returned, and the final SIP non-success status can be retrieved from `err.status` as shown above.

### Canceling an INVITE we sent

Sometimes, we may want to cancel an INVITE that we have sent before it is answered.  Related to this, we may simply want to get access to details of the actual INVITE that was sent out over the wire.  To do this, we can provide a `cbRequest` callback in the callback object mentioned above.  This callback receives a `req` object representing the INVITE that was sent over the wire.  If we later want to cancel the INVITE, we simply call `req.cancel()`.

```js
let invite, dlg;
srf.createUac('sip:1234@10.10.100.1', {localSdp: mySdp}, {
    cbRequest: (err, req) => invite = req),
    cbProvisional: (res) => console.log(`got provisional response: ${res.status}`))
  })
  .then((dialog) => {
    dlg = dialog
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err}`);
  });

  someEmitter.on('some-event', () => {
    // something happened to make us want to cancel the call
    if (!dlg && invite) invite.cancel();
  });
```

### Authentication

If we send an INVITE that is challenged (with either a 401 Unauthorized or a 407 Proxy Authentication Required), we can have the drachtio framework handle this if we provide the username and password in the `opts.auth` parameter:

```js
let mySdp; // populated somehow with SDP we want to offer
srf.createUac('sip:1234@10.10.100.1', {
  localSdp: mySdp,
  auth: {
    username: 'dhorton',
    password: 'foobar'
  }
})
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```
### Sending through an outbound SIP proxy

If we want the INVITE to be sent through an outbound SIP proxy, rather than directly to the endpoint specified in the Request-URI, we can specify an `opts.proxy` parameter:

```js
let mySdp; // populated somehow with SDP we want to offer
srf.createUac('sip:1234@10.10.100.1', {
  localSdp: mySdp,
  proxy: 'sip:proxy.enterprise.com'
})
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```

### Specifying calling number or called number

If we want to specify the calling party number to use in the From header of the INVITE, and/or the called party number to use in the To header as well as the Request-URI, we can do so simply like this:

```js
let mySdp; // populated somehow with SDP we want to offer
srf.createUac('sip:1234@10.10.100.1', {
  localSdp: mySdp,
  callingNumber: '+18584083089',
  calledNumber: '+15083345988'
})
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```
> Pro tip: Where possible, use this approach of providing 'opts.callingNumber' rather than trying to provide a full From header.

> Pro tip: If you *really* want to provide the full From header, for the host part of the uri use the string 'localhost'.  The drachtio server will handle this by replacing the host value with the proper IP address for the server.

### Sending a 3PCC INVITE

A scenario known as third-party call control (3PCC) occurs when a UAC sends an INVITE with no body -- i.e., no session description protocol is initially offered.  In this call flow, after the offer is received in the 200 OK response from the B party, the UAC sends its sdp in the ACK to establish the media flows.

To accomplish this, the [Srf#createUAC](/docs/api#Srf+createUAC) can be used.  However, because of the need to specify an SDP in the ACK, the application must take additional responsibility for generating the ACK.

Furthermore, instead of delivering a Dialog, the Promise (or callback) will render an object containing two properties:
* `sdp`: the sdp received in the 200 OK from the B party, and
* `ack`: a function that the application must call, providing the SDP to be included in the ACK as the first parameter.  The function returns a Promise that resolves to the Dialog created by the ACK.

With that as background, let's see an example:

```js
srf.createUac('sip:1234@10.10.100.1', {
  callingNumber: '+18584083089',
  calledNumber: '+15083345988',
  noAck: true
})
  .then((obj) => {
    console.log(`received sdp offer ${obj.sdp} from B party`);
    let mySdp = allocateSdpSomehow();
    return obj.ack(mySdp);
  })
  .then((dialog) => {
    dialog.on('destroy', () => console.log('called party ended call'));  
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```
Note a few things in this example:
* There was no `opts.body` parameter in the call to [Srf#createUAC](/docs/api#Srf+createUAC): the absence of a body signals the drachtio framework that we are intending a 3PCC scenario.
* The use of `opts.noAck` indicates that we do *not* want the framework to generate the ACK for us, and that instead we will explicitly call the `obj.ack()` function to take responsibility for that.

There is also a fairly common alternative 3PCC special case where you may want to offer a "null" SDP in the ACK, i.e. creating an initially inactive media stream that you will later activate with a re-INVITE.  In that case, you can simply remove the `opts.noAck` parameter and the Promise/callback will deliver the completed dialog as in the normal case -- the framework will generate the appropriate "null" sdp and generate the ACK for you.

```js
srf.createUac('sip:1234@10.10.100.1', {
  callingNumber: '+18584083089',
  calledNumber: '+15083345988'
})
  .then((dialog) => {
    console.log(`created dialog with inactive media`);
    dialog.on('destroy', () => console.log('called party ended call'));  
  })
  .catch((err) => {
    console.log(`call failed with ${err});
  });
```
## B2BUA

When we receive an incoming INVITE (the A leg), and then send a new outgoing INVITE on a different SIP transaction (the B leg), we are acting as a back to back user agent. We use the [Srf#createB2BUA](/docs/api#Srf+createB2BUA) method to accomplish this.  Once again, either a callback approach or a Promise-based approach is supported.

As with the UAC scenario, the simplest usage is to provide the Request-URI to send the B leg to and the sdp to offer on the B leg.  If successful, our application receives two SIP dialogs: a UAS dialog (the A leg) and a UAC dialog (the B leg);

```js
srf.invite((req, res) => {
  srf.createB2BUA(req, res, 'sip:1234@10.10.100.1', {localSdpB: req.body})
    .then({uas, uac} => {
      console.log('call successfully connected');

      // when one side hangs up, we hang up the other
      uas.on('destroy', () => uac.destroy());
      uac.on('destroy', () => uas.destroy());
    })
    .catch((err) => console.log(`call failed to connect: ${err}`));
});
```
Beyond this simple example, there are many options.  Let's look at some of them:

### Copying headers from the A leg onto the B leg
It's quite common for us to want to include on the B leg INVITE some of the headers that we received on the A leg; vice-versa, we may want to include on the A leg response some of the headers that we received on the B leg response.  

This can be achieved with the `opts.proxyRequestHeaders` and the `opts.proxyResponseHeaders` properties in the optional `opts` parameter.  If provided, these should include an array of header names that should be copied from one to the other.

The example below illustrates a B2BUA app that wants to pass authentication headers between endpoints 

```js
srf.invite((req, res) => {
  srf.createB2BUA(req, res, 'sip:1234@10.10.100.1' {
    localSdpB: req.body,
    proxyRequestHeaders: ['Proxy-Authorization', 'Authorization'],
    proxyResponseHeaders: ['WWW-Authenticate', 'Proxy-Authentication']
  })
    .then({uas, uac} => {
      console.log('call successfully connected');

      // when one side hangs up, we hang up the other
      uas.on('destroy', () => uac.destroy());
      uac.on('destroy', () => uas.destroy());
    })
    .catch((err) => console.log(`call failed to connect: ${err}`));
});
```

### Obtaining the UAC dialog as soon as possible
When the [Srf#createB2BUA](/docs/api#Srf+createB2BUA) completes successfully, it provides us the two dialog that have been established.  

However, in rare cases it may be desirable to receive the UAC dialog as soon as it is established -- that is, as soon as we have received a 200 OK from the B party, before we have sent the 200 OK back to the A party, and before the Srf#createB2BUA](/docs/api#Srf+createB2BUA) method has resolved the Promise that it returns.

For this need, similar to [Srf#createUAC](/docs/api#Srf+createUAC), there is an optional callback object that contains a callback named `cbFinalizedUac` that, if provided, is invoked with the UAC dialog as soon as it is created.  (Note: the `cbProvisional` and `cbRequest` callbacks are also available).

```js
srf.invite((req, res) => {
  srf.createB2BUA(req, res, 'sip:1234@10.10.100.1', {
    localSdpB: req.body
  }, {
    cbFinalizedUac: (uac) => {
      console.log(`successfully connected to B party at ${uac.remote.contact}`);
    }
  })
    .then({uas, uac} => {
      console.log('call successfully connected');

      // when one side hangs up, we hang up the other
      uas.on('destroy', () => uac.destroy());
      uac.on('destroy', () => uas.destroy());
    })
    .catch((err) => console.log(`call failed to connect: ${err}`));
});
```
### Modifying SDP offered to A leg in SIP response

By default, [Srf#createUAC](/docs/api#Srf+createUAC) will respond with a 200 OK to the A leg INVITE with the sdp that it received from the B party in the 200 OK on the B leg.  In other words, it simply proxies the session description protocol offer from B back to A.

Sometimes, however, it is desirable to transform or modify the SDP received on the B leg before sending it back on the A leg.  For this purpose, the `opts.localSdpA` parameter is available.  This parameter can either be a string, containing the sdp to offer in the 200 OK back to the A leg, or it can be a function returning a Promise that resolves to the sdp to return in the 200 OK to the A leg.  The function has the signature `(sdpB, res)`, where `sdpB` is the session description offer we received in the 200 OK from the B party, and `res` is the response object we received on the B leg.

Rather than include an example here, please refer to the source code for the [drachtio-b2b-media-proxy](https://github.com/davehorton/drachtio-b2b-media-proxy/blob/master/app.js) application, which is a simple B2BUA that uses [rtpengine](https://github.com/sipwise/rtpengine) to proxy the media.  This is a great example of when you would want to transform the SDP from B before returning the final session description offer to A.

### Choosing not to propagate failure from B leg 

By default, if we get a final SIP non-success from the B party it will be propagated back to the A party.  There are times where we would prefer not to do so; for instance, if having failed to connect the A party to one endpoint or phone number, we would now wish to try another.

Setting `opts.passFailure` to value of false enables this behavior.

```js
srf.invite((req, res) => {
  srf.createB2BUA(req, res, 'sip:1234@10.10.100.1', {localSdpB: req.body, passFailure: false})
    .then({uas, uac} => {
      console.log('call connected to primary destination');
    })
    .catch((err) => {
      // try backup if we got a sip non-success response and the caller did not hang up
      if (err.status !== 487) {
        console.log(`failed connecting to primary, will try backup: ${err}`);
        srf.createB2BUA(req, res, 'sip:1234@10.10.100.2', {
          localSdpB: req.body}
        })
        .then({uas, uac} => {
          console.log('call connected to backup destination');
        })
        catch((err) => {
          console.log(`failed connecting to backup uri: ${err}`);
        });
      }
    });
  });
```
Note that we had to check that the reason for the failure connecting our first attempt was not a 487 Request Cancelled, because this is the error we receive when the caller (A party) hung up before we connected the B party.  In that case, we no longer would want to attempt a backup destination.

This also answers a related question you may have had: what happens when the A part hangs up before connected, and does our app need to do anything specifically to cancel the B leg when the A leg cancels?  The answer to the latter is no, the drachtio framework will automatically cancel the B leg if the A leg is canceled.

## Subscribe Dialogs

SUBSCRIBE requests also establish SIP Dialogs, as per [RFC 3265](https://www.ietf.org/rfc/rfc3265.txt). 

A UAC (the _subscriber_) sends a SUBSCRIBE request with an Event header indicating the event that is being subscribed to, and the UAS (the _notifier_) responds with a 202 Accepted response.  The UAS should then immediately send a NOTIFY to the UAC of the current state of the requested resource.  To terminate a SUBSCRIBE dialog, the UAS sends a NOTIFY request with Subscription-State: terminated; while a UAC would send another SUBSCRIBE with an Expires: 0.

The dialog produced by the [Srf#createUAS](/docs/api#Srf+createUAS) method will be a SUBSCRIBE dialog if the request was a SUBSCRIBE.  While the [Srf#createUAS](/docs/api#Srf+createUAS) method call will send a 202 Accepted, it does *not* send the initial NOTIFY request that should follow -- the application must do that, since the content can only be determined by the application itself.

```js
srf.subscribe((req, res) => {
  srf.createUAS(req, res, {
    {
      headers: {'Expires': req.get('Expires')
    }  
  })
    .then((dialog) => {
      dialog.on('destroy', () => console.log('remote party terminated dialog'));
      
      // send initial NOTIFY
      let myContent = 'some content reflecting current resource state..';

      return dialog.request({
        method: 'NOTIFY',
        headers: {
          'Subscription-State': 'active',
          'Event': req.get('Event'),
          'Content-Type': 'application/pidf+xml' // or whatever
        },
        body: myContent
      });
    });
});
```
> Pro tip: If you need to query a dialog to see whether it is an INVITE or a SUBSCRIBE dialog, you can use the `dialogType` (read-only) property of the Dialog object to determine that.

The [Srf#createUAC](/docs/api#Srf+createUAC) method can also be used to generate a SUBSCRIBE dialog as a UAC/subscriber.  To do this, specify `opts.method` should be set to 'SUBSCRIBE'.

```js
srf.createUac('sip:resource@example.com', {
    method: 'SUBSCRIBE',
    headers: {
      'From': '<sip:user@locahost>',
      'Event': 'presence',
      'Expires': 3600
    }
  })
  .then((dialog) => {
    dialog.on('destroy', () => console.log('remote party ended dialog'));

    dialog.on('notify', (req, res) => {
      res.send(200);

      console.log(`received NOTIFY for event ${req.get('Event')}`);
      if (req.body) {
        console.log(`received content of type ${req.get('Content-Type')}: ${req.body}`);
      }
    });
  });
```
> Note in the example above the use of 'localhost' as the host part of the uri of the From header.  As mentioned earlier, this will cause the drachtio server to replace this with the appropriate IP address of the server.

