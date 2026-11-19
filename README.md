# Project description

Mulesoft Anypoint project created for Mulesoft training Anypoint Platform Development Fundamentals (DEX401) training.

# [Walkthrough 2-3] Connecting Anypoint Studio to Salesforce with OAuth 2.0 JWT Bearer
 
This guide describes how to configure the **Salesforce Config** in a MuleSoft Anypoint Studio project using the **OAuth 2.0 JWT Bearer** flow, with an **External Client App** in the Salesforce org.
 
## Why not Basic Authentication?
 
Basic Authentication (username + password + security token) uses the SOAP `login()` call. Newer Salesforce orgs (including Trailhead Playgrounds) block it. Symptoms:
 
- Studio shows: `INVALID_LOGIN: Invalid username, password, security token; or user locked out.`
- *Setup → Login History* shows: `Failed: SOAP Login Disabled`
In that case no combination of credentials will work, so use OAuth instead.
 
## Prerequisites
 
- Anypoint Studio with a Mule project that uses the Salesforce connector
- A Salesforce org (Developer Edition / Trailhead Playground) and a user with System Administrator rights
- Java `keytool` available on the command line (it ships with the JDK)
> **Note:** A Trailhead Playground is a Developer Edition org, not a sandbox. Do **not** use `test.salesforce.com`. Use your org's My Domain URL.
 
---
 
## Step 0: Find your My Domain URL
 
1. In Salesforce go to *Setup → My Domain*.
2. Copy **Current My Domain URL**, e.g.:
```
<my-domain-name>.trailblaze.my.salesforce.com
```
 
   (Trailhead orgs use a partitioned domain, so it is not simply `<name>.my.salesforce.com`.)
 
You will need it for the token endpoint in Step 4.
 
## Step 1: Create a keystore and certificate
 
Run in a terminal:
 
```bash
keytool -genkeypair -alias sfjwt -keyalg RSA -keysize 2048 -validity 365 -keystore sf-jwt.jks -storepass <STORE_PASSWORD>
 
keytool -exportcert -alias sfjwt -keystore sf-jwt.jks -storepass <STORE_PASSWORD> -rfc -file sf-jwt.crt
```
 
- `keytool` asks for a name and organization details. Any values are fine. Confirm with `yes`.
- At the end it prints no summary message. That is normal.
- The values you chose are your configuration values for Step 4:
  - **Alias:** `sfjwt`
  - **Store password:** `<STORE_PASSWORD>` (the key password is the same)
- Result: `sf-jwt.jks` (private key + certificate, stays on the Mule side) and `sf-jwt.crt` (public certificate, uploaded to Salesforce).
Verify the keystore:
 
```bash
keytool -list -keystore sf-jwt.jks -storepass <STORE_PASSWORD>
```
 
It should list one entry `sfjwt` of type `PrivateKeyEntry`.
 
Copy `sf-jwt.jks` to the project's `src/main/resources` folder.
 
> **Security:** Never commit the `.jks` file or the real password to the repository. Add `sf-jwt.jks` to `.gitignore` and keep the password in a secure properties file or environment variable. In this guide the password is a placeholder.
 
## Step 2: Create the External Client App in Salesforce
 
New orgs often offer only the **External Client App Manager** (not classic Connected Apps).
 
1. *Setup → External Client App Manager → New External Client App*.
2. Fill in the basic info (name, contact email).
3. In **OAuth Settings**:
   - Check **Enable OAuth**
   - Callback URL: any placeholder, e.g. `https://localhost/callback`
   - Selected scopes: `Manage user data via APIs (api)` and `Perform requests at any time (refresh_token, offline_access)`
4. Under **Flow Enablement**, check **Enable JWT Bearer Flow** and upload `sf-jwt.crt` (the `.crt` file, not the `.jks`).
   - Leave **Issue JSON Web Token (JWT)-based access tokens for named users** unchecked. It is unrelated.
5. Save the app.
## Step 3: Pre-authorize your user
 
1. Open the app in *Setup → External Client App Manager*, tab **Policies**.
2. Under OAuth policies, set **Permitted Users** to **Admin approved users are pre-authorized**, then save.
3. Assign your user to the app, via one of these routes:
   - **Policies tab:** add your profile (e.g. *System Administrator*) or a permission set, if the picker is available.
   - **Permission set:** *Setup → Permission Sets → New* (e.g. `Mule Integration`), open it, go to **External Client App Access** (may be labeled *Connected App Access* / *Assigned Connected Apps*), add your app to the enabled list, then **Manage Assignments → Add Assignment** and select your user.
Without this step the login fails with an error about the user not having approved the consumer.
 
## Step 4: Get the Consumer Key
 
1. Open the app, tab **Settings → OAuth Settings**.
2. Click **Consumer Key and Secret** and verify your identity with the emailed code.
3. Copy the **Consumer Key**. The Consumer Secret is not needed for JWT.
## Step 5: Configure the Salesforce Config in Anypoint Studio
 
Open the Salesforce Config element, set the connection type to **OAuth JWT** (instead of Basic Authentication) and fill in:
 
| Field | Value |
|---|---|
| Consumer key | Consumer Key from Step 4 |
| Key store path | `sf-jwt.jks` (file in `src/main/resources`) |
| Store password | `<STORE_PASSWORD>` |
| Alias / certificate alias | `sfjwt` |
| Key password (if present) | `<STORE_PASSWORD>` |
| Principal | The Salesforce username, `<firstname.lastname>@<org-domain>.com` |
| Token endpoint | `https://<my-domain-name>.trailblaze.my.salesforce.com/services/oauth2/token` |
| Audience URL | `https://login.salesforce.com` (if rejected, try your My Domain URL) |
 
Click **Test Connection**.
 
## Troubleshooting
 
| Symptom | Likely cause / fix |
|---|---|
| `ToolingNotAvailableException: Tooling instance is not available` | Studio's tooling process is not running. Use *Window → Preferences → Anypoint Studio → Tooling → Restart Studio Services*, or restart Studio. This error never reaches Salesforce, so it is unrelated to credentials. |
| `INVALID_LOGIN` with Basic Authentication | SOAP login disabled in the org. Check *Setup → Login History* for `SOAP Login Disabled`. Use this JWT guide. |
| `user hasn't approved this consumer` | Step 3 not done or not yet propagated. Recheck the assignment and wait a minute or two. |
| `invalid_grant` (signature / audience) | The certificate uploaded to Salesforce does not match the keystore, or the audience is wrong. Re-export `sf-jwt.crt` from the current `.jks` and re-upload it, and try the other audience value. |
| Nothing obvious | *Setup → Login History* shows the exact reason of the failed attempt. If no entry appears, the request is reaching a different org or endpoint. |
 
If you regenerate the keystore, always export and upload a **new** `.crt`, because the old one no longer matches.
 
