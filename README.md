# apigee-auth0-external-authorization

## Summary
This proxy demonstrates how to enable external authorization with Auth0.  It saves the authorization code generated by Auth0 in Edge so that the proxy can subsequently validate that authorization code on the /token request.

The `auth0-oauth` proxy does not save the JWT as an external access token.  Once the client has the JWT, then they should include that as an `Authorization: Bearer` header on subsequent requests.  All of your other proxies should validate the JWT, with the public certificate, expiry and custom claims.  Make sure to include the [JWT/JWE/JWS Java Callout](https://github.com/apigee/iloveapis2015-jwt-jwe-jws) written by Dino, which validates JWTs.

### High-level Flow
* `/authorize` - validates the client ID redirect uri and forwards the request to Auth0 to generate the authorization code.
* `/redirect` - called by Auth0 and extracts the Auth0 authorization code and saves it in Apigee Edge.
* `/token` - validates the client ID and redirect URI and forward the request to Auth0 to generate the JWT.

## Prerequisites
You may need to [install acurl](http://docs.apigee.com/api-services/content/using-oauth2-security-apigee-edge-management-api#howtogetoauth2tokens).


You must export the following environment variables:
```
export ae_password=apigeepassword
export ae_username=apigeeusername
export ae_org=apigeeorg
```

### Auth0
You should create an account in [Auth0](https://auth0.com/) and create a [Auth0 client](https://auth0.com/docs/clients) to obtain a client ID and secret.
A good starting point is to read this [community article](https://community.apigee.com/articles/42269/auth0-with-apigee.html) if you are not familiar with Auth0. It will provides instructions to setup a new client and obtain a client Id and secret.

* The redirect URI that you should enter into your Auth0 client is `https://org-env.apigee.net/oauth_auth0/redirect`.
* Create a user in Auth0. This is the user that you will use to login during the redirect step.

### Update edge.json
You should update the following entries in the `auth0-oauth/edge.json` file. Just use a find and replace to update all the values.  
* `yourdomain.auth0.com`
* `https://yourdomain.auth0.com`
* `https://org-env.apigee.net/`

## Helpful to Know
You don't have to review this links, since I list all the commands to deploy the proxy.  But if you are interested in learning more about the config and deploy plugins, then read these links.  
* [apigee-config-maven-plugin](https://github.com/apigee/apigee-config-maven-plugin)
* [apigee-maven-deploy-plugin](https://github.com/apigee/apigee-deploy-maven-plugin)

## Deploy
Follow the steps below to deploy shared flows, the proxy, create the developer, product and app in Edge.

### Deploy a Shared Flows
The shared flows must be deployed first.

```
cd auth0-ProxyDefaultFaultRule
mvn install -PtestSharedFlow -Dusername=$ae_username -Dpassword=$ae_password -Dorg=$ae_org -Dauthtype=oauth
cd ../auth0-ProxyFaultRules
mvn install -PtestSharedFlow -Dusername=$ae_username -Dpassword=$ae_password -Dorg=$ae_org -Dauthtype=oauth
```

### Proxy
This will deploy the `auth0-auth` proxy and create the Apigee developer named `john@example.com`, a product named `auth0-product` and an app named `auth0-app`.  
```
cd ../auth0-auth
mvn install -Ptest -Dusername=$ae_username -Dpassword=$ae_password \
                    -Dorg=$ae_org -Dauthtype=oauth -Dapigee.config.options=create
```
## Create the client ID and secret in Edge
Once you create the client in Auth0 and create the Apigee product and app, then you have to add the client ID and secret into Apigee Edge. Use the following API calls to add the credentials to Apigee Edge.

[Create a consumer key/secret](http://docs.apigee.com/management/apis/post/organizations/%7Borg_name%7D/developers/%7Bdeveloper_email_or_id%7D/apps/%7Bapp_name%7D/keys/create)

Use the developer `john@example.com` and the app named `auth0-app`.

[Associate the consumer key and secret with an Apigee product](http://docs.apigee.com/management/apis/post/organizations/%7Borg_name%7D/developers/%7Bdeveloper_email_or_id%7D/apps/%7Bapp_name%7D/keys/%7Bconsumer_key%7D)

The payload for this request is shown below.
```
{ "apiProducts": ["auth0-product"] }
```


## Test the Deployment
After you have your Auth0 configured and you have deployed the proxy you should copy the following command into your browser.  Be sure to update the `{org}`, `{env}` and `{clientID}`.
```
https://{org}-{env}.apigee.net/oauth_auth0/authorize?{client_id}=clientID&response_type=code&redirect_uri=https://callback.io&scope=openid
```

1. You will be redirected to the Auth0 login screen.
  * login with the Auth0 user that you created.  
2. The authorization code will be downloaded to your local machine.
3. Copy the authorization code into the request below. Be sure to update
  * `org`
  * `env`
  * `clientID`
  * `secret`

```
curl -X POST \
  https://{org}-{env}.apigee.net/oauth_auth0/token \
  -H 'cache-control: no-cache' \
  -H 'content-type: application/x-www-form-urlencoded' \
  -d 'client_id={clientID}&code={authCode}&grant_type=authorization_code&redirect_uri=https%3A%2F%2Fcallback.io&client_secret={secret}'
```

Now you have a Auth0 JWT that was proxied through Apigee Edge.  
Please note the following:
1. The JWT is not stored in Apigee Edge (TODO create separate repo to demo this)
2. You should validate the JWT with Dino's [JWT Java callout](https://github.com/apigee/iloveapis2015-jwt-jwe-jws).
3. You will miss out on some analytics because the Verify API key/Validate Access Token policy is not included in the proxy, so you won't know who the developer is.
   * However, the JWT includes the client ID, so you could extract that from the JWT and then include a VerifyAPIKey policy on the preflow, they Apigee will know who the developer is and will populate all the analytics associated with the developer.  

## TODO
Create a new repo that demonstrates how to save the JWT as an access token in Apigee Edge. 
