# shopify-flutterflow-oauth-guide
A complete, step-by-step master reference guide to implementing native mobile OAuth 2.0 with Shopify's Customer Account API inside FlutterFlow without broken web redirects.
---

## Architecture Overview

Instead of traditional web redirects that break in mobile app wrappers, this architecture opens a secure native sandbox sheet on the device using `flutter_web_auth_2`. The app programmatically intercepts the custom mobile URI scheme (`shop.{SHOP_ID}.app://callback`) before it attempts to load an actual web page, shuts the browser down instantly, extracts the authorization code, and passes it to Shopify's OAuth token exchange API.

---

## Placeholders Reference

Replace the following placeholders across the guide with your own store credentials:

| Placeholder | Description | Example |
| --- | --- | --- |
| `YOUR_SHOP_ID` | Numeric Shopify Shop ID | `12345678900` |
| `YOUR_CLIENT_ID` | Shopify Customer Account API Client ID | `abcd1234-5678-90ef-ghij-klmnopqrstuv` |
| `YOUR_STORE_DOMAIN` | Your custom or standard store domain | `account.yourstore.com` |

---

## Step 1: Shopify Admin Dashboard Configuration

1. Log into your **Shopify Partner / Admin Dashboard** and navigate to your store settings.


2. Go to **Customer Account API Settings**.


3. Set the **Client Type** dropdown to:
`Public (mobile app)`

4. In the **Allowed Callback URLs** list box, add this exact string character-for-character:


```text
shop.YOUR_SHOP_ID.app://callback

```


5. Save your changes on the Shopify Dashboard.



---

## Step 2: FlutterFlow App Settings & State Management

### 1. Turn On Custom Authentication

1. Go to **Settings & Integrations (Gear icon) > App Settings > Authentication**.


2. Toggle **Enable Authentication** to **ON**.


3. Set **Authentication Type** to **Custom**.


4. Set your **Initial Page** (e.g., `LoginPage`) and your **Logged In Page** (e.g., `HomeDashboard`).



### 2. Define Local App State Variables

Navigate to the **App State** menu on the left sidebar and create two fields:

* **Field 1:**
* **Field Name:** `customerAccessToken`

* **Data Type:** `String`

* **Persisted:** `ON`



* **Field 2:**
* **Field Name:** `customerRefreshToken`

* **Data Type:** `String`

* **Persisted:** `ON`




---

## Step 3: Custom Function (URL Builder)

Prevents FlutterFlow's visual engine from double-encoding parameters into broken web text (e.g., changing `://` to `%253A%252F%252F`).

1. Go to **Custom Code (`</>`) > + Add > Function**.


2. **Function Name:** `buildShopifyLoginUrl`

3. **Return Type:** `String` (Ensure *Is List* is `OFF`)


4. Paste this code between the `MODIFY CODE` comments:



```dart
String? buildShopifyLoginUrl() {
  /// MODIFY CODE ONLY BELOW THIS LINE
  // Generate a clean current UNIX timestamp string locally
  final String timestamp = (DateTime.now().millisecondsSinceEpoch / 1000).toString();

  // Construct the baseline string using pre-encoded target pieces
  // to avoid FlutterFlow's double-encoding engine wrapper!
  return "https://YOUR_STORE_DOMAIN/authentication/oauth/authorize"
      "?client_id=YOUR_CLIENT_ID"
      "&response_type=code"
      "&redirect_uri=shop.YOUR_SHOP_ID.app%3A%2F%2Fcallback" // Single-encoded
      "&scope=openid%20email%20customer-account-api%3Afull"
      "&code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
      "&code_challenge_method=S256"
      "&prompt=login"
      "&state=$timestamp"
      "&nonce=$timestamp";
  /// MODIFY CODE ONLY ABOVE THIS LINE
}

```

5. Click **Save Function** and **Check Code** to compile.



---

## Step 4: Custom Action (Browser Interceptor)

Spins up the native browser sandbox sheet and intercepts the custom URI callback.

1. Go to **Custom Code (`</>`) > + Add > Action**.


2. **Action Name:** `launchInAppLoginBrowser`

3. **Define Arguments:**
* `urlString` (`String`)




4. **Action Properties:**
* **Return Value:** `String` (Nullable checked)




5. **Pubspec Dependencies:**
* `flutter_web_auth_2: ^3.0.0`



6. Paste the following action code:



```dart
// Automatic FlutterFlow imports
import '/flutter_flow/flutter_flow_theme.dart';
import '/flutter_flow/flutter_flow_util.dart';
import '/custom_code/actions/index.dart';
import '/flutter_flow/custom_functions.dart';
import 'package:flutter/material.dart';
// Begin custom action code
// DO NOT REMOVE OR MODIFY THE CODE ABOVE!
import 'package:flutter_web_auth_2/flutter_web_auth_2.dart';

Future<String?> launchInAppLoginBrowser(String urlString) async {
  try {
    // The scheme is the exact text string before the ://
    final result = await FlutterWebAuth2.authenticate(
      url: urlString,
      callbackUrlScheme: "shop.YOUR_SHOP_ID.app",
      options: const FlutterWebAuth2Options(
        preferEphemeral: true,
      ),
    );

    final Uri callbackUri = Uri.parse(result);
    final String? authCode = callbackUri.queryParameters['code'];
    return authCode;
  } catch (e) {
    print("Authentication intercept failed or window closed: $e");
    return null;
  }
}

```

7. Click **Save Action** and **Check Code** to compile.



---

## Step 5: Shopify Token Exchange API Setup

Trades the intercepted authorization code for customer account tokens.

1. Go to **API Calls** on the left menu and select or add your Shopify Token Exchange API configuration.


2. **Configuration Settings:**
* **Method Type:** `POST`

* **API URL:** `https://YOUR_STORE_DOMAIN/authentication/oauth/token`



3. **Variables Tab:**
* Declare variable: `receivedCode` | Type: `String`



4. **Body Tab:**
* Format: `x-www-form-urlencoded`

* Map the key-value pairs:


* `grant_type` $\rightarrow$ `authorization_code`

* `client_id` $\rightarrow$ `YOUR_CLIENT_ID`

* `code` $\rightarrow$ `[receivedCode]` (Variable)


* `redirect_uri` $\rightarrow$ `shop.YOUR_SHOP_ID.app://callback`





5. Click **Save**.



---

## Step 6: UI Action Flow (Login Button)

Attach this flow directly to your main **Login Button** inside the Action Flow Editor:

* **Action 1: Fire Interceptor Browser**

* Custom Action: `launchInAppLoginBrowser`

* `urlString` parameter: Select `Custom Functions > buildShopifyLoginUrl`

* **Action Output Variable Name:** `extractedCode`



* **Action 2: Add Conditional Gate**

* **First Value:** `Action Outputs > extractedCode`

* **Rule:** Change from *Equal To* to `Is Set`



* **TRUE Path (Code Received):**
* **Action 2A: Backend API Call**

* API Call: `Shopify TokenExchange`

* Variable `receivedCode` $\rightarrow$ `Action Outputs > extractedCode`

* **Output Variable Name:** `tokenResponse`



* **Action 2B: Save Tokens to App State**

* Update App State:
* Field: `customerAccessToken` $\rightarrow$ `Action Outputs > tokenResponse > JSON Body > $.accessToken`

* Field: `customerRefreshToken` $\rightarrow$ `Action Outputs > tokenResponse > JSON Body > $.refreshToken`





* **Action 2C: Route to Home Storefront**

* Target Page: `HomeDashboard`

* **Allow Back Navigation:** `OFF`





* **FALSE Path (User Cancelled/Swiped down):**
* Leave completely empty (graceful exit).





---

## Step 7: Page On-Load Housekeeping

* Clear the `LoginPage` **On Page Load** actions of legacy link parameters or web redirect code.


* *(Optional)* Implement a page load check: If App State `customerAccessToken` **Is Set**, execute an immediate `Navigate To` straight to `HomeDashboard` to bypass login for returning customers.


* Delete old web `CallbackPage` files from the project tree entirely.



---

## Troubleshooting Checklist

* **Browser stays open on a white screen or shows 400 Mismatch Error:**
Shopify's Allowed Callback URLs list doesn't match character-for-character. Verify that `shop.YOUR_SHOP_ID.app://callback` matches identically in Shopify Admin, your custom action scheme string, and your API request body.


* **Window closes, but app drops back onto the login page:**
Double-check that your Conditional Action is using `Is Set` (not `Equal To`), and verify that your App State JSON paths use exact camelCase (`$.accessToken` and `$.refreshToken`).


* **Stuck or erratic behavior in emulator tests:**
Emulators (e.g., BlueStacks) do not support security deep intent registers reliably. Execute testing exclusively using physical Android or iOS smartphones.
