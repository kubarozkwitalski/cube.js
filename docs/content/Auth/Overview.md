---
title: Overview
permalink: /security
category: Authentication & Authorization
menuOrder: 1
---

In Cube.js, authorization (or access control) is based on the **security
context**. The diagram below shows how it works during the request processing in
Cube.js:

<p
  style="text-align: center"
>
  <img
  src="https://raw.githubusercontent.com/statsbotco/cube.js/master/docs/content/authentication-overview.png"
  style="border: none"
  width="80%"
  />
</p>

Authentication is handled outside of Cube.js. A typical use case would be:

1. A web server serves an HTML page containing the Cube.js client, which needs
   to communicate securely with the Cube.js API.
2. The web server should generate a JWT with an expiry to achieve this. The
   server could include the token in the HTML it serves or provide the token to
   the frontend via an XHR request, which is then stored it in local storage or
   a cookie.
3. The JavaScript client is initialized using this token, and includes it in
   calls to the Cube.js API.
4. The token is received by Cube.js, and verified using any available JWKS (if
   configured)
5. Once decoded, the token claims are injected into the [security
   context][ref-sec-ctx].

<!-- prettier-ignore-start -->
[[info |]]
| **In development mode, the token is not required for authorization**, but you
| can still use it to [pass a security context][ref-sec-ctx].
<!-- prettier-ignore-end -->

## Using JSON Web Key Sets (JWKS)

### Configuration

As mentioned previously, Cube.js supports verifying JWTs using industry-standard
JWKS. The JWKS can be provided either from a URL, or as a JSON object conforming
to [JWK specification RFC 7517 Section 4][link-jwk-ref], encoded as a string.

#### Using a key as a JSON string

Add the following to your `cube.js` configuration file:

```javascript
module.exports = {
  jwt: {
    key: '<JWKS_AS_STRING>',
  },
};
```

Or configure the same using environment variables:

```dotenv
CUBEJS_JWK_KEY='<JWKS_AS_STRING>'
```

#### Using a key from a URL

<!-- prettier-ignore-start -->
[[info |]]
| When using a URL to fetch the JWKS, Cube.js will automatically cache the
| response, re-use it and update if a key rotation has occurred.
<!-- prettier-ignore-end -->

Add the following to your `cube.js` configuration file:

```javascript
module.exports = {
  jwt: {
    jwkUrl: '<URL_TO_JWKS_JSON>',
  },
};
```

Or configure the same using environment variables:

```dotenv
CUBEJS_JWK_URL='<URL_TO_JWKS_JSON>'
```

### Verifying claims

Cube.js can also verify the audience, subject and issuer claims in JWTs.
Similarly to JWK configuration, these can also be configured in the `cube.js`
configuration file:

```javascript
module.exports = {
  jwt: {
    audience: '<AUDIENCE_FROM_IDENTITY_PROVIDER>',
    issuer: ['<ISSUER_FROM_IDENTITY_PROVIDER>'],
    subject: '<SUBJECT_FROM_IDENTITY_PROVIDER>',
  },
};
```

Using environment variables:

```dotenv
CUBEJS_JWT_AUDIENCE='<AUDIENCE_FROM_IDENTITY_PROVIDER>'
CUBEJS_JWT_ISSUER='<ISSUER_FROM_IDENTITY_PROVIDER>'
CUBEJS_JWT_SUBJECT='<SUBJECT_FROM_IDENTITY_PROVIDER>'
```

### Custom claims namespace

Cube.js can also extract claims defined in custom namespaces. Simply specify the
namespace in your `cube.js` configuration file:

```javascript
module.exports = {
  jwt: {
    claimsNamespace: 'my-custom-namespace',
  },
};
```

## Generating Tokens

Authentication tokens are generated based on your API secret. Cube.js CLI
generates an API Secret when a project is scaffolded and saves this value in the
`.env` file as `CUBEJS_API_SECRET`.

You can generate two types of tokens:

- Without security context, which will mean that all users will have the same
  data access permissions.
- With security context, which will allow you to implement role-based security
  models where users will have different levels of access to data.

<!-- prettier-ignore-start -->
[[info |]]
| It is considered best practice to use an `exp` expiration claim to limit the
| lifetime of your public tokens. [Learn more in the JWT docs][link-jwt-docs].
<!-- prettier-ignore-end -->

You can find a library to generate JWTs for your programming language
[here][link-jwt-libs].

In Node.js, the following code shows how to generate a token which will expire
in 30 days. We recommend using the `jsonwebtoken` package for this.

```javascript
const jwt = require('jsonwebtoken');
const CUBE_API_SECRET = 'secret';

const cubejsToken = jwt.sign({}, CUBE_API_SECRET, { expiresIn: '30d' });
```

Then, in a web server or cloud function, create a route which generates and
returns a token. In general, you will want to protect the URL that generates
your token using your own user authentication and authorization:

```javascript
app.use((req, res, next) => {
  if (!req.user) {
    res.redirect('/login');
    return;
  }
  next();
});

app.get('/auth/cubejs-token', (req, res) => {
  res.json({
    // Take note: cubejs expects the JWT payload to contain an object!
    token: jwt.sign(req.user, process.env.CUBEJS_API_SECRET, {
      expiresIn: '1d',
    }),
  });
});
```

Then, on the client side, (assuming the user is signed in), fetch a token from
the web server:

```javascript
let apiTokenPromise;

const cubejsApi = cubejs(
  () => {
    if (!apiTokenPromise) {
      apiTokenPromise = fetch(`${API_URL}/auth/cubejs-token`)
        .then((res) => res.json())
        .then((r) => r.token);
    }
    return apiTokenPromise;
  },
  {
    apiUrl: `${API_URL}/cubejs-api/v1`,
  }
);
```

You can optionally store this token in local storage or in a cookie, so that you
can then use it to query the Cube.js API.

[link-jwt-docs]:
  https://github.com/auth0/node-jsonwebtoken#token-expiration-exp-claim
[link-jwt-libs]: https://jwt.io/#libraries-io
[link-jwk-ref]: https://tools.ietf.org/html/rfc7517#section-4
[ref-sec-ctx]: /security/context
