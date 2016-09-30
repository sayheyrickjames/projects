# About Threads

## 1.1 Introduction

Threads


Is a visual design language for our users that outlines individual component and layout specifications. It currently covers the Plaid Marketing site (plaid.com).

It is intended to be a living document that will be updated as we continue to develop the tenets and specifics of our system. It functions as a point of record, as well as our central place for design collaboration.

## 1.2 Goals

Define our visual language


Incorporate everything we’ve learned on all sides of our business into one cohesive point of view describing how we visually speak and articulate our value proposition to our customers.

Develop a single system


Provide a unified experience across platforms and device sizes, for all internal stakeholders to easily build, maintain and iterate.


# Glossary

Most Plaid API calls involve three pieces of information: a `client_id`, `secret`, and an `access_token`. All three of these are considered private information - they are not meant to be shared or hard coded directly into an app or site. This poses a problem for Plaid Link, as a user will be onboarding directly in the browser. To address this, Plaid client accounts now have an additional identifier: a `public_key`.

## The public_key

1. Associates users that onboard through Plaid Link with your `client_id` and `secret`
2. Cannot be used to make authenticated Plaid API calls (i.e. retrieving account and routing numbers for a user)

## The public_token

Once a user has successfully onboarded via Plaid Link, the module will provide a `public_token`. This is in contrast to typical Plaid API requests, which return an `access_token`.

1. Is safe to expose in an app or browser
2. Cannot be used to retrieve account and routing numbers (for auth) or transactions (for connect) directly
3. Can be exchanged for a Plaid `access_token` via the `/exchange_token` endpoint.

The [getting started](#getting-started) guide below illustrates how to use your `public_key` to get up and running with Plaid Link. The guide also covers how to exchange a `public_token` for an `access_token`.


# Getting started

There are two different integrations:

1. [Simple](#step-2-simple-integration) The simple integration provides a customizable "Link your Bank Account" button and handles submitting a Plaid `public_token` to a server endpoint that you specify.
2. [Custom](#step-2-custom-integration) The custom integration lets you trigger the Plaid Link module via client-side Javascript. You specify your own callback function to be called with the `public_token` once a user has authenticated.


## Step 1: Get your public_key

Your `public_key` is available from the [Plaid dashboard][4]:

<figure class="quickstart-image">
  [![Plaid Dashboard](/images/docs/link-key-screenshot.png)](https://dashboard.plaid.com/)
</figure>

Your `public_key` is a less privileged version of your `client_id` and `secret`. It simply associates accounts you create using Plaid Link with your `client_id`. All Plaid API requests **must** be made using your private `client_id` and `secret`.

## Step 2: Simple integration

Include the following markup (the `<form>` and `<script>` tags) in your site or web application:

```html
<!-- A hidden input named public_token will be appended to this form
once the user has completed the Link flow. Link will then submit the
form, sending the public_token to your server. -->
<form id="some-id" method="POST" action="/authenticate"></form>

<!-- To use Link with longtail institutions on Connect, set the
data-longtail attribute to 'true'. See the Parameter Reference for
additional documentation. -->
<script
  src="https://cdn.plaid.com/link/stable/link-initialize.js"
  data-client-name="Client Name"
  data-form-id="some-id"
  data-key="test_key"
  data-product="auth"
  data-env="tartan">
</script>
```

See the [parameter reference](#simple-integration) for complete documentation on possible configurations and instructions for enabling longtail institutions on Connect.

The injected "Link your Bank Account" button has the ID `plaid-link-button` and can be styled with CSS:

```css
#plaid-link-button {
  border: 10px solid pink;
}
```

## Step 2: Custom integration

The custom integration allows you to decide, with a few lines of client-side Javascript, when the module is triggered. You can trigger the "Institution Select" view or trigger a particular institution's credentials form. See below:

```html
<button id='linkButton'>Open Link - Institution Select</button>
<button id='bofaButton'>Open Link - Bank of America</button>
<script src="https://cdn.plaid.com/link/stable/link-initialize.js"></script>
<script>
  var linkHandler = Plaid.create({
    env: 'tartan',
    clientName: 'Client Name',
    key: 'test_key',
    product: 'auth',
    // To use Link with longtail institutions on Connect, set the
    // 'longtail' option to true:
    // longtail: true,
    onLoad: function() {
      // The Link module finished loading.
    },
    onSuccess: function(public_token, metadata) {
      // Send the public_token to your app server here.
      // The metadata object contains info about the institution the
      // user selected and the account ID, if selectAccount is enabled.
    },
    onExit: function() {
      // The user exited the Link flow.
    }
  });

  // Trigger the BofA login view
  document.getElementById('bofaButton').onclick = function() {
    linkHandler.open('bofa');
  };

  // Trigger the standard institution select view
  document.getElementById('linkButton').onclick = function() {
    linkHandler.open();
  };
</script>
```

See the [**parameter reference**](#custom-integration) for complete documentation on possible configurations and instructions for enabling longtail institutions on Connect.

`Plaid.create` accepts one argument, a configuration `Object`, and returns an `Object` with one function, `open`, and one property, `institutions`. `open` accepts either no arguments or an optional [institution type][7]. If no argument is provided, the "Institution Select" view is opened. If a valid institution type is provided, the login form for that particular institution is opened. The exposed `institutions` property is an `Array` of `Object`s in the form:

```javascript
[{
  name: 'Bank of America',
  type: 'bofa',
  auth: true,
  connect: true
},
...]
```

The `institutions` property will be populated with all supported institutions for a given product. That is, the list of institutions will be different for `auth` and `connect`. Use the `institutions` property to dynamically generate a list of supported institutions for your Link integration - by doing so, your app will support new institutions and products automatically.

## Step 3: Write server-side handler

The Link module handles the entire onboarding flow securely and quickly but does not actually retrieve account or transaction data for a user. Instead, the Link module returns a `public_token` that is safe to expose in the browser.

This `public_token` must be exchanged for a Plaid `access_token` using the [`/exchange_token`](#-exchange_token-endpoint) API endpoint. To do so, you must add a server side handler. A sample node.js server-side handler is provided below - full application samples can be found in the [Examples](#examples) section below.

With the [simple integration](#step-2-simple-integration), the `public_token` will be appended to the form specified by `data-form-id` as a hidden input. This form will be automatically submitted once a user has completed onboarding.

With the [custom integration](#step-2-custom-integration), the `callback` you specified in `Plaid.create` will be called with one argument, the `public_token`. It is your responsibility to send it to your app server.

In the example above, the form `some-id` had `action="/authenticate"`. So we'll add a `/authenticate` route to our server side code that expects a `POST` request with a field named `public_token` in the request body. We'll then use that `public_token` along with our private `client_id` and `secret` to retrieve a Plaid `access_token` (via the [`/exchange_token` endpoint](#-exchange_token-endpoint)). This `access_token` should be saved in a database and used to retrieve account and transaction data. Below is a sample server-side handler using [Express](http://expressjs.com) and the [plaid-node](https://github.com/plaid/plaid-node) library:

```javascript
var express = require('express');
var plaid = require('plaid');

var app = express();

var plaidClient = new plaid.Client(process.env.PLAID_CLIENT_ID,
                                   process.env.PLAID_SECRET,
                                   plaid.environments.tartan);

app.post('/authenticate', function(req, res) {
  var public_token = req.body.public_token;

  // Exchange a public_token for a Plaid access_token
  plaidClient.exchangeToken(public_token, function(err, exchangeTokenRes) {
    if (err != null) {
      // Handle error!
    } else {
      // This is your Plaid access token - store somewhere persistent
      // The access_token can be used to make Plaid API calls to
      // retrieve accounts and transactions
      var access_token = exchangeTokenRes.access_token;

      plaidClient.getAuthUser(access_token, function(err, authRes) {
        if (err != null) {
          // Handle error!
        } else {
          // An array of accounts for this user, containing account
          // names, balances, and account and routing numbers.
          var accounts = authRes.accounts;

          // Return account data
          res.json({accounts: accounts});
        }
      });
    }
  });
});
```

Check out the [Examples](#examples) for more sample code and complete example apps.

## Step 4: Test with sandbox credentials

The Link module has a sandbox mode that works with the [Plaid API sandbox][3]. To enable the sandbox, set `data-key` or `key` to `test_key` (for simple and custom integrations, respectively). This lets you see the flow for each individual institution Plaid supports, including the multi-factor authentication process when applicable.

### For simple integrations:

```html
<!-- A hidden input named public_token will be appended to this form
once the user has completed the Link flow. Link will then submit the
form, sending the public_token to your server. -->
<form id="some-id" method="POST" action="/authenticate"></form>

<script
  src="https://cdn.plaid.com/link/stable/link-initialize.js"
  data-client-name="Client Name"
  data-form-id="some-id"
  data-key="test_key"
  data-product="auth"
  data-env="tartan">
</script>

```

### For custom integrations:

```html
<button id="plaidLinkButton">Open Plaid Link</button>

<script src="https://cdn.plaid.com/link/stable/link-initialize.js"></script>
<script>
var sandboxHandler = Plaid.create({
  env: 'tartan',
  clientName: 'Client Name',
  key: 'test_key',
  product: 'auth',
  onSuccess: function(public_token, metadata) {
    console.log(public_token, metadata);
  },
});

document.getElementById('plaidLinkButton').onclick = function() {
  // Trigger the "Institution Select" view
  sandboxHandler.open();
};
</script>
```

If you are having trouble using the module in sandbox mode, check the developer console in your browser for error messages.

# Updating existing accounts

When a user changes their username, password, or MFA credentials with a financial institution or is locked out of their account, they must update their credentials with Plaid as well.

Link's **update mode** makes this process secure and painless and is available in both the simple and custom integrations. To use **update mode**, initialize Link with the `public_token` for the user you wish to update.

For the simple integration add the `data-token` attribute as follows:

```html
<!-- A hidden input named public_token will be appended to this form
once the user has completed the Link flow. Link will then submit the
form, sending the new public_token to your server. -->
<form id="some-id" method="POST" action="/authenticate"></form>

<script
  src="https://cdn.plaid.com/link/stable/link-initialize.js"
  data-client-name="Client Name"
  data-form-id="some-id"
  data-key="test_key"
  data-product="auth"
  data-env="tartan"
  data-token="test,chase,connected">
</script>
```

The custom initializer takes a similarly-named `token` added to the parameter hash:

```html
<script src="https://cdn.plaid.com/link/stable/link-initialize.js"></script>
<script>
var linkHandler = Plaid.create({
  env: 'tartan',
  clientName: 'Client Name',
  key: 'test_key',
  product: 'auth',
  token: 'test,chase,connected',  // <- token to patch
  onLoad: function() {
    // The Link module finished loading.
  },
  onSuccess: function(public_token, metadata) {
    // Send the public_token to your app server here.
    // The metadata object contains info about the institution the
    // user selected and the account ID, if selectAccount is enabled.
  },
  onExit: function() {
    // The user exited the Link flow.
  },
});

// Trigger the authentication view
document.getElementById('linkButton').onclick = function() {
  linkHandler.open();
};
</script>
```

Link will jump directly to the login view for the appropriate institution when in update mode. Note that for custom integrations calling Link with an institution will not work as usual:
```
linkHandler.open('chase');
```

Instead it will function identically to:
```
linkHandler.open();
```
This is because the user's public token is associated with a particular institution and it does not make sense to open another institution's authentication.


## Test update mode with sandbox tokens

For update mode a suitable sandbox public token can be generated by inserting the desired [institution type](https://plaid.com/docs/api#institutions) into the string `test,{institution_type},connected`.

For example:
```
test,chase,connected
test,usaa,connected
test,wells,connected
```


# Reference

## /exchange_token Endpoint

The `/exchange_token` endpoint is available in both the tartan
(`https://tartan.plaid.com`) and production (`https://api.plaid.com`) environments.

| Method | Endpoint          | Required Parameters                   | Optional Parameters |
|--------|-------------------|---------------------------------------|---------------------|
| POST   | `/exchange_token` | `client_id`, `secret`, `public_token` | `account_id`        |

The `/exchange_token` endpoint has already been integrated into the [plaid-node](https://github.com/plaid/plaid-node), [plaid-go](https://github.com/plaid/plaid-go), [plaid-ruby](https://github.com/plaid/plaid-ruby), and [plaid-python](https://github.com/plaid/plaid-python) client libraries. Support for [plaid-java](https://github.com/plaid/plaid-java) is coming soon.

If you are working with a library that does not yet support the `/exchange_token` endpoint you can simply make a standard HTTP request:

```console
$ curl -X POST https://tartan.plaid.com/exchange_token \
>   -d client_id="$plaid_client_id" \
>   -d secret="$plaid_secret" \
>   -d public_token="$public_token_from_plaid_link_module"
```

For a valid request, the API will return a JSON response similar to:

```json
{
  "access_token": "foobar_plaid_access_token"
}
```
For possible error codes, see the full listing of [Plaid error codes](https://github.com/plaid/support/blob/master/errors.md).

## Simple integration

| Parameter          | Required  | Description                                                                                                                                                                                             |
|--------------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `data-client-name` | required  | Displayed once a user has successfully linked their account.                                                                                                                                            |
| `data-form-id`     | required  | The DOM ID associated with form that the Link module will append the `public_key` to as a hidden input and submit when a user completes the onboarding flow.                                            |
| `data-product`     | required  | The Plaid product you wish to use: `auth`, `connect`, `info`, `risk`, or `income`.                                                                                                                                          |
| `data-key`         | required  | The `public_key` associated with your account; available form the [dashboard][4].                                                                                                                       |
| `data-env`         | required  | The Plaid API environment on which to create user accounts. For development and testing, use `tartan`. For production use, use `production`.<br /><br />**Note:** all `production` requests are billed. |
| `data-webhook`     | optional  | Specify a [webhook](https://plaid.com/docs/api#webhook) to associate with a user.                                                                                                                           |
| `data-token`       | optional  | Specify an existing user's public token to launch Link in update mode. This will cause Link to open directly to the authentication step for that user's institution.                                   |
| `data-longtail`    | optional  |  Set to `true` to launch Link with [longtail institution support][8] enabled. Longtail institutions are only available with Connect (`data-product` is `connect`).<br /><br />**Note:** Your account **must** be enabled for longtail usage - [drop us a note](https://plaid.com/contact/sales) to gain access.

## Custom integration

| Parameter       | Required | Description                                                                                                                                                                                             |
|--------------   |-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `clientName`    | required  | Displayed once a user has successfully linked their account.                                                                                                                                            |
| `product`       | required  | The Plaid product you wish to use: `auth`, `connect`, `info`, `risk`, or `income`.                                                                                                                                           |
| `key`           | required  | The `public_key` associated with your account; available form the [dashboard][4].                                                                                                                       |
| `env`           | required  | The Plaid API environment on which to create user accounts. For development and testing, use `tartan`. For production use, use `production`.<br /><br />**Note:** all `production` requests are billed. |
| `onSuccess`     | required  | A function that is called when a user has successfully onboarded their account. The function should expect two arguments, the `public_key` and a [`metadata` object](#onsuccess-metadata).                                                              |
| `onExit`        | optional  | A function that is called when a user has specifically exited the Link flow.                                                                                                                            |
| `onLoad`        | optional  | A function that is called when the Link module has finished loading. Calls to `plaidLinkHandler.open()` prior to the onLoad callback will be delayed until the module is fully loaded.                  |
| `webhook`       | optional  | Specify a [webhook](https://plaid.com/docs/api#webhook) to associate with a user.                                                                                                                           |
| `token`         | optional  | Specify an existing user's public token to launch Link in update mode. This will cause Link to open directly to the authentication step for that user's institution.                                   |
| `longtail`      | optional  |  Set to `true` to launch Link with [longtail institution support][8] enabled. Longtail institutions are only available with Connect (`product` is `connect`).<br /><br />**Note:** Your account **must** be enabled for longtail usage - [drop us a note](https://plaid.com/contact/sales) to gain access.
| `selectAccount` | optional  |  Set to `true` to launch Link with the 'Select Account' pane enabled. This allows users to select an individual account once they've authenticated. <br /><br />The Plaid Account ID of this account is provided in the `metadata` object of the `onSuccess` callback under the key `account_id`.<br /><br />**Note:** If you are using Link with Connect, all accounts will be listed on the Select Account pane. If you are using Link with Auth, only depository accounts will be listed.

## onSuccess Metadata

The `onSuccess` callback should take two arguments, the `public_key` and a `metadata` object. The `metadata` object provides the following information:

| Parameter     | Description                                                                                                                                                      |
|---------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `institution` | An Object with two properties:<br/> - `name`: The full institution name, such as 'Bank of America'.<br />- `type`: The institution type, such as 'bofa'.|
| `account_id`  | The Plaid Account ID of the account selected by the user.<br /><br />**Note:** Only applicable when the Select Account pane is enabled.|


# Security

All data sent to Plaid is encrypted using a secure HTTPS connection with our servers. While you can use the module on a non-HTTPS site (all data sent to Plaid operates in an entirely different context and connection), we strongly recommend using HTTPS at all times.

More information about Plaid's security policies and practices can be found on our [website][5].


# Browser support

## Desktop

<ul class="link-docs-compatibility-table">
  <li>
    ![Google Chrome](/images/docs/chrome.png)
    <span>Fully supported</span>
  </li>
  <li>
    ![Firefox](/images/docs/firefox.png)
    <span>Fully supported</span>
  </li>
  <li>
    ![Safari](/images/docs/safari.png)
    <span>Fully supported</span>
  </li>
  <li>
    ![Internet Explorer](/images/docs/ie10.png) 10
    <span>Fully supported</span>
  </li>
  <li>
    ![Internet Explorer](/images/docs/ie9.png) 9
    <span>Fully supported &#42;</span>
  </li>
  <li>
    ![Internet Explorer](/images/docs/ie8.png) 8
    <span class="not-supported">Not supported</span>
  </li>

  <p class="link-docs-compatibility-table__caveat"> 	&#42;You must host the Link module on a HTTPS site for compatibility due to IE9 security restrictions.
  </p>
</ul>

## Mobile

Modern mobile browsers are supported, including most iPhone and Android devices. If you encounter any inconsistencies, [open an issue][6].


# Expansion

Plaid Link is currently a web-only integration. Plaid Link will offer native iOS and Android bindings in the coming months.


# Support and questions

Find answers to many common integration questions and concerns at our [Support Center][9].

If you're still stuck, reach out to [support@plaid.com](mailto:support@plaid.com) with information describing the issue that you're experiencing and we'll get back to you as soon as we can.

# Examples

Check out our code samples and community-contributed resources to kickstart your Plaid Link integration!

### Client Libraries

Link provides a drop-in HTML snippet for the client-side integration but does requires a server-side handler to coordinate exchanging a Link `public_token` for a Plaid `access_token` via the [`/exchange_token` endpoint](#-exchange_token-endpoint).

The `/exchange_token` endpoint is integrated into each of our client libraries. Check out some examples:

- [plaid-node](https://github.com/plaid/plaid-node#examples)
- [plaid-go](https://github.com/plaid/plaid-go#exchange-a-plaid-link-public_token-for-an-access_token)
- [plaid-ruby](https://github.com/plaid/plaid-ruby#exchanging-a-link-public_token-for-a-plaid-access_token)
- [plaid-python](https://github.com/plaid/plaid-python#exchange)
- [plaid-java](https://github.com/plaid/plaid-java#exchange-a-plaid-link-public_token-for-an-api-access_token)

### Sum App

We built [Sum](https://demo.plaid.com) to demonstrate a sample Link client and server-side integration using our [client libraries](#client-libraries). Check out the source code in the language or framework of your choice:
- [node.js](https://github.com/plaid/link/tree/master/examples/node)
- [go](https://github.com/plaid/link/tree/master/examples/go)
- [ruby](https://github.com/plaid/link/tree/master/examples/ruby)
- [RequireJS](https://github.com/plaid/link/tree/master/examples/requirejs)
- [UIWebView](https://github.com/plaid/link/tree/master/examples/uiwebview)
- python (coming soon!)

Each implementation has a complete README with instructions for running the app
locally!

### Community Resources

- [React component](https://github.com/pbernasconi/react-plaid-link), by [@pbernasconi](https://github.com/pbernasconi)
- [Ember component](https://github.com/jasonkriss/ember-plaid), by [@jasonkriss](https://github.com/jasonkriss)
- [Angular component](https://github.com/csbarnes/angular-plaid-link), by [@csbarnes](https://github.com/csbarnes)
- [Example Angular/Ionic app](https://github.com/pbernasconi/plaid-link-ionic-example), by [@pbernasconi](https://github.com/pbernasconi)


[1]: https://plaid.com/docs
[2]: https://demo.plaid.com
[3]: https://plaid.com/docs/api#sandbox
[4]: https://plaid.com/account/
[5]: https://plaid.com/security
[6]: https://github.com/plaid/link/issues/new
[7]: https://plaid.com/docs/api#institutions
[8]: https://plaid.com/docs/api#long-tail-institutions
[9]: https://support.plaid.com
