# Middleware

drachtio-srf is a middleware framework.  As we saw above, we handle SIP INVITEs using `srf.invite(handler)` where our handler function is invoked with `(req, res)` and the arguments provided are objects that represent the incoming SIP request and the SIP response the application will send, respectively.  

All of the SIP methods are routed similarly, e.g.
```js
srf.register((req, res) => {...handle REGISTERs});

srf.options((req, res) => {...handle OPTIONS});

srf.subscribe((req, res) => {...handle SUBSCRIBE}); //...etc
```

drachtio middleware can also be installed via the `.use` method.  The middleware can be globally applied to all requests, or can be scoped by method.  Below is an example where we use global middleware to log all requests, and a second middleware that parses authentication credentials from incoming REGISTER requests.
```js
const Srf = require('drachtio-srf');
const srf = new Srf();
const registrationParser = require('drachtio-mw-registration-parser');

srf.use((req, res, next) => console.log(`incoming ${req.method from ${req.source_address}}`));
srf.use('register', registrationParser);

srf.register((req, res) => {
  // middleware has populated req.registration
  console.log(`registration info: ${req.registration});

    // {
    //    type: 'register' or 'unregister'
    //    expires: expires value in either Contact or Expires header
    //    contact: sip contact / address to send requests to
    //    aor: address-of-record being registered
    // } ;

});
```
Example middleware include:
* [drachtio-mw-digest-auth](https://www.npmjs.com/package/drachtio-mw-digest-auth) - implements server-side digest authentication, per [RFC 2617](https://www.ietf.org/rfc/rfc2617.txt)
* [drachtio-mw-registration-parser](https://www.npmjs.com/package/drachtio-mw-registration-parser)
