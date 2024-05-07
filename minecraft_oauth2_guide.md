# HOW TO DO OAUTH2 CODE FLOW FOR MINECRAFT AUTH

This is a simplified version of [this Wiki.vg page](https://wiki.vg/Microsoft_Authentication_Scheme). This guide assumes you can read HTTP requests and know how to do them.

## Step 1: Getting a client ID

You will need to do this to be able to access Microsoft, Xbox, and Minecraft APIs.



1. Sign up for [Azure](https://azure.com).
2. Register an Entra ID domain (formerly Active Directory).
3. Add an app registration to your domain.
   - Set it for personal accounts only.
   - You do not need to add a redirect URI for now.
4. Under Manage > Authentication, enable the following:
   - Allow public client flows
   - Live SDK support
5. Fill out the form at <https://aka.ms/mce-reviewappid>
> [!IMPORTANT]  
> You **must** do this step and get approved in order to use Minecraft APIs!
6. Copy client ID for said application.

## Step 2: OAuth2 token
We'll be using the device code flow documented [here](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code). You can implement other methods yourself.

1. Send the following HTTP request:
```
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode
Content-Type: application/x-www-form-urlencoded

client_id=${CLIENT_ID}
&scope=XboxLive.signin%20offline_access
```
> [!IMPORTANT]  
> Those scopes are **required** to get a Minecraft bearer.
2. Parse the response as a JSON object.
3. Send the user to the `verification_url` and provide them with `user_code`, and/or send them directly to `verification_url` with `?otc=${user_code}` appended.
4. Respecting the polling rate of `interval` seconds, send the following request:
```
POST https://login.microsoftonline.com/consumers/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:device_code&client_id=${CLIENT_ID}
&device_code=${step_3_response.device_code}
```
> [!NOTE]  
> The above request hardcodes the tenant of `consumers`. This is intentional, as you cannot request an XSTS token (see later) using any other tenant.
5. Parse the response as JSON, and do the following for each response with `error`:
   - `authorization_pending`: Poll again (repeat step 4). You're waiting on the user still.
   - `authorization_declined`: User declined to authenticate. Do not continue to poll.
   - `bad_verification_code`: Halt and Catch Fire. Your program is failing to parse the `device_code` and send it properly.
   - `expired_token`: User took too long to authenticate. Do not continue to poll.
   - no `error`: Proceed to step 6.
6. Parse the authentication response. Note `expires_in`, the expiry time, `access_token`, your access token, and `refresh_token`, your refresh token (see addendum I.)
## Step 3: Getting an Xbox Live token

1. Send the following HTTP request:
```
POST https://user.auth.xboxlive.com/user/authenticate
Content-Type: application/json
Accept: application/json

{
    "Properties": {
        "AuthMethod": "RPS",
        "SiteName": "user.auth.xboxlive.com",
        "RpsTicket": "d=${access_token}"
    },
    "RelyingParty": "http://auth.xboxlive.com",
    "TokenType": "JWT"
 }
```
2. Parse the response, noting the `Token`, and the userhash at `DisplayClaims.xui[0].uhs`.
## Step 4: Getting the XSTS token
1. Send the following HTTP request:
```
POST https://xsts.auth.xboxlive.com/xsts/authorize
Content-Type: application/json
Accept: application/json

{
   "Properties": {
       "SandboxId": "RETAIL",
       "UserTokens": [
           "${xbox_Token}"
       ]
   },
   "RelyingParty": "rp://api.minecraftservices.com/",
   "TokenType": "JWT"
}
```
2. Parse the response, noting the `Token` (which is now an XSTS token).
> [!WARNING]  
> The response may also contain an `XErr` value. See addendum II for the common ones. 
## Step 5: Getting the bearer
Finally almost there!
```
POST https://api.minecraftservices.com/authentication/login_with_xbox
Content-Type: application/json

{
    "identityToken": "XBL3.0 x=${userhash};${xbox_XSTS}"
}
```
This endpoint returns an `access_token` and when it `expires_in`.  

## Done
Congratulations! You can now use the Minecraft APIs with your own homegrown authentication.

## Addendums
### I - refresh an account
```
POST https://login.microsoftonline.com/consumers/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id=${CLIENT_ID}
&scope=XboxLive.signin%20offline_access
&refresh_token=${refresh_token}
&grant_type=refresh_token
```
There will be either be:
- an `error`, `error_codes[]`, and `error_description`
- or a new `refresh_token` and `access_token`

### II - XErr meanings
Taken from [here](https://wiki.vg/Microsoft_Authentication_Scheme#Obtain_XSTS_token_for_Minecraft).

- `2148916233`: The account doesn't have an Xbox account.
- `2148916235`: The account is from a country where Xbox Live is not available/banned.
- `2148916236`: The account needs adult verification on the Xbox page. (South Korea)
- `2148916237`: The account needs adult verification on the Xbox page. (South Korea)
- `2148916238`: The account is a child (under 18) and cannot proceed unless the account is added to a Family by an adult.
