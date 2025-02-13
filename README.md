<a href="https://zerodha.tech"><img src="https://zerodha.tech/static/images/github-badge.svg" align="right" /></a>

# OTP Gateway

OTP (One Time Password) Gateway is a standalone web server that provides a central gateway to verify user addresses such as e-mails and phone numbers, and to send 2FA confirmations to such addresses.

- Use the built in web UI to easily integrate with existing applications.
- Use the HTTP/JSON APIs to build your own UI.
- Basic multi-tenancy with namespace+secret BasicAuth for seggregating applications.

![address](https://user-images.githubusercontent.com/547147/52076261-501e1300-25b4-11e9-8641-2189d0e4afb7.png)
![otp](https://user-images.githubusercontent.com/547147/51735115-7d4a5d00-20ac-11e9-8a86-3985665a7820.png)
![email-otp](https://user-images.githubusercontent.com/547147/51734344-407d6680-20aa-11e9-8e8e-03db29d8f900.png)


### Built-in providers
- SMTP
- AWS Pinpoint SMS
- Kaleyra SMS, WhatsApp


### Webhook providers
Any external provider can be integrated by defining one or more [webhook providers in the config](https://github.com/knadh/otpgateway/blob/745ce8fb9d3491a8774d5290006691fded560fa4/config.sample.toml#L141). A JSON payload is posted to the webhook endpoint whenever an OTP is generated.


# How does it work?

The application is agnostic of the user's "address" and the OTP / verification codes. These are handled by providers.

- Addresses are strings, for example, e-mail IDs, phone numbers, bank account numbers etc.
- OTPs are also just strings, for instance, 6 digit codes sent as SMSes or a penny value dropped to a bank account.

The gateway sends the OTP value to the user's address using a provider (an upstream that takes the OTP + message and sends it to the user) and the user then has to read the OTP and enter it on the gateway's web view to complete the verification.

# Usage

Download the latest release from the [releases page](https://github.com/knadh/otpgateway/releases). 

- Copy config.sample.toml to config.toml and edit the configuration.
- Run `./otpgateway`
- Refer to the [API reference](#user-content-api-reference) to send OTPs.

### Built in UI

1. Generate an OTP for a user server side in your application:
   `curl -u "myAppName:mySecret" -X PUT -d "to=john@doe.com&provider=smtp" localhost:9000/api/otp/uniqueIDForJohnDoe`
2. Use the `OTPGateway()` Javascript function (see the Javascript plugin section) to initiate the modal UI on your webpage. On receiving the Javascript callback, post it back to your application and confirm that the OTP is indeed verified:
   `curl -u "myAppName:mySecret" -X POST localhost:9000/api/otp/uniqueIDForJohnDoe/status`

### Your own UI

Use the APIs described below to build your own UI.

# API reference

### List providers

`curl -u "myAppName:mySecret" localhost:9000/api/providers`

```json
{
  "status": "success",
  "data": ["smtp"]
}
```

### Initiate an OTP for a user

```shell
curl -u "myAppName:mySecret" -X PUT -d "to=john@doe.com&provider=smtp&extra={\"yes\": true}" localhost:9000/api/otp/uniqueIDForJohnDoe
```

| param               | description                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| :id                 | (optional) A unique ID for the user being verified. If this is not provided, an random ID is generated and returned. It's good to send this as a permanent ID for your existing users to prevent users from indefinitely trying to generate OTPs. For instance, if your user's ID is 123 and you're verifying the user's e-mail, a simple ID can be MD5("email.123"). _Important_. The ID is only unique per namespace and not per provider. |
| provider            | ID of the provider plugin to use for verification. The bundled e-mail provider's ID is "smtp".                                                                                                                                                                                                                                                                                                                                               |
| to                  | (optional) The address of the user to verify, for instance, an e-mail ID for the "smtp" provider. If this is left blank, a view is displayed to collect the address from the user.                                                                                                                                                                                                                                                           |
| channel_description | (optional) Description to show to the user on the OTP verification page. If not provided, it'll show the default description or help text from the provider plugin.                                                                                                                                                                                                                                                                            |
| address_description | (optional) Description to show to the user on the address collection page. If not provided, it'll show the default description or help text from the provider plugin.                                                                                                                                                                                                                                                                          |
| otp                 | (optional) The OTP or code to send to the user for verification. If not provided, a random OTP is generated and sent                                                                                                                                                                                                                                                                                                                   |
| ttl                 | (optional) OTP expiry in seconds. If not provided, the default value from the config is used. |
| max_attempts        | (optional) Maximum number of OTP verification attempts. If not provided, the default value from the config is used. |
| skip_delete         | (optional) After a successful OTP verification, the OTP is deleted. If this is set true `true`, OTP is not deleted and is let to expire gradually. |
| extra               | (optional) An extra payload (JSON string) that will be returned with the OTP                                                                                                                                                                                                                                                                                                                                                                 |

```json
{
  "status": "success",
  "data": {
    "namespace": "myAppName",
    "id": "uniqueIDForJohnDoe",
    "to": "john@doe.com",
    "channel_description": "",
    "address_description": "",
    "extra": { "yes": true },
    "provider": "smtp",
    "otp": "354965",
    "max_attempts": 5,
    "attempts": 5,
    "closed": false,
    "ttl": 300,
    "url": "http://localhost:9000/otp/myAppName/uniqueIDForJohnDoe"
  }
}
```

### Validate an OTP entered by the user

Every incorrect validation here increments the attempts before further attempts are blocked.
Once the OTP is verified, it is deleted, unless `skip_delete=true` is passed in the params.
`curl -u "myAppName:mySecret" -X POST -d "action=check&otp=354965" localhost:9000/api/otp/uniqueIDForJohnDoe`

```json
{
  "status": "success",
  "data": {
    "namespace": "myAppName",
    "id": "uniqueIDForJohnDoe",
    "to": "john@doe.com",
    "channel_description": "",
    "address_description": "",
    "extra": { "yes": true },
    "provider": "smtp",
    "otp": "354965",
    "max_attempts": 5,
    "attempts": 5,
    "closed": false,
    "ttl": 300,
    "url": "http://localhost:9000/otp/myAppName/uniqueIDForJohnDoe"
  }
}
```

### Check whether an OTP request is verified

This is used to confirm verification after a callback from the built in UI flow.
`curl -u "myAppName:mySecret" -X POST localhost:9000/api/otp/uniqueIDForJohnDoe/status`

```json
{
  "status": "success",
  "data": {
    "namespace": "myAppName",
    "id": "uniqueIDForJohnDoe",
    "to": "john@doe.com",
    "channel_description": "",
    "address_description": "",
    "extra": { "yes": true },
    "provider": "smtp",
    "otp": "354965",
    "max_attempts": 5,
    "attempts": 5,
    "closed": false,
    "ttl": 300
  }
}
```

or an error such as

```json
{ "status": "error", "message": "OTP not verified" }
```

# Javascript plugin

The gateway comes with a Javascript plugin that enables easy integration of the verification UI into existing applications. Once a server side call to generate an OTP is made and a namespace and id are obtained, calling `OTPGateway()` opens the verification UI in a modal popup. Upon completion of verification by the user, a callback is triggered.

```html
<!-- The id #otpgateway-script is required for the script to work //-->
<script
  id="otpgateway-script"
  src="http://localhost:9000/static/otp.js"
></script>
<script>
  // 1. Make a call to the server to generate and send an OTP and return the
  // the :namespace and :id for the OTP.
  // 2. Invoke the verification UI for the user with the namespace and id values,
  // and a callback which is triggered when the user finishes the flow.
  OTPGateway(
    namespaceVal,
    idVal,
    function(nm, id) {
      console.log("finished", nm, id);

      // 3. Post the namespace and id to your server that will make the
      // status request to the gateway and on success, update the user's
      // address in your records as it's now verified.
    },
    function() {
      console.log("cancelled");
    }
  );
</script>
```

Licensed under the MIT license.
