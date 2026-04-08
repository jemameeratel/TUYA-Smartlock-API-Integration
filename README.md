### Tuya OpenAPI: Offline Temporary Password (v1.1)

This documentation covers the two-step process required to generate an offline passkey for a Tuya-compatible smart lock: Authenticaton and Passkey Generation.

#### 1. Global Variables & Credentials

Before making any requests, ensure your environment has these variables defined:

Plaintext

```
<BASE_URL>      = https://openapi.tuyaus.com (or your specific region: eu, sg, etc.)
<CLIENT_ID>     = Your Tuya Cloud Access ID / Client ID
<CLIENT_SECRET> = Your Tuya Cloud Access Secret
<DEVICE_ID>     = The specific virtual ID of the smart lock
<TIMESTAMP>     = Standard 13-digit Unix timestamp (milliseconds)
```

---

#### 2. Authentication: Get Access Token

All requests to the Tuya API require a valid Access Token, which is valid for 2 hours.

**Endpoint:**

HTTP

```
GET <BASE_URL>/v1.0/token?grant_type=1
```

**Required Headers:**

HTTP

```
client_id: <CLIENT_ID>
sign: <CALCULATED_SIGNATURE>
t: <TIMESTAMP>
sign_method: HMAC-SHA256
```

*Note on Signature Calculation for Token Request:* The signature (`sign`) is an `HMAC-SHA256` hash of your Client ID, the timestamp, and the request details, signed with your Client Secret. `sign = HMAC-SHA256( <CLIENT_ID> + <TIMESTAMP> + "GET\n" + HASH_SHA256("") + "\n\n/v1.0/token?grant_type=1", <CLIENT_SECRET> )`

**Expected Response (Success):**

JSON

```
{
  "result": {
    "access_token": "<ACCESS_TOKEN>",
    "expire_time": 7200,
    "refresh_token": "<REFRESH_TOKEN>",
    "uid": "<TUYA_USER_ID>"
  },
  "success": true,
  "t": <TIMESTAMP>
}
```

---

#### 3. Generate Offline Temporary Password (v1.1)

This endpoint generates a mathematical offline PIN code that does not require the lock to be connected to Wi-Fi at the time of entry.

**Endpoint:**

HTTP

```
POST <BASE_URL>/v1.1/devices/<DEVICE_ID>/door-lock/offline-temp-password
```

**Required Headers:**

HTTP

```
client_id: <CLIENT_ID>
access_token: <ACCESS_TOKEN>
sign: <CALCULATED_SIGNATURE_WITH_TOKEN>
t: <TIMESTAMP>
sign_method: HMAC-SHA256
Content-Type: application/json
```

**Request Body (Payload):** The v1.1 API requires a flat JSON object. Timestamps must be in 10-digit Unix format (seconds).

JSON

```
{
  "effective_time": <START_TIMESTAMP_SECONDS>,
  "invalid_time": <END_TIMESTAMP_SECONDS>,
  "type": "multiple",
  "name": "<GUEST_IDENTIFIER>"
}
```

*Note on times:* For offline passwords, `effective_time` and `invalid_time` should typically be rounded down to the nearest hour depending on the lock's specific firmware requirements.

*Note on Signature Calculation for POST Request:* `sign = HMAC-SHA256( <CLIENT_ID> + <ACCESS_TOKEN> + <TIMESTAMP> + "POST\n" + HASH_SHA256(<JSON_BODY_STRING>) + "\n\n/v1.1/devices/<DEVICE_ID>/door-lock/offline-temp-password", <CLIENT_SECRET> )`

**Expected Response (Success):**

JSON

```
{
  "result": {
    "offline_temp_password": "<10_DIGIT_PIN_CODE>"
  },
  "success": true,
  "t": <TIMESTAMP>
}
```

---

#### 4. Error Handling

If a request fails, Tuya will return `success: false` along with an error code.

**Common Error Syntax:**

JSON

```
{
  "code": "<ERROR_CODE>",
  "msg": "<ERROR_DESCRIPTION>",
  "success": false,
  "t": <TIMESTAMP>
}
```

- `1004`: Sign Invalid (Usually a timestamp mismatch or incorrect hashing).
  
- `1010`: Token Expired (Request a new token using the `/v1.0/token` endpoint).
  
- `1106`: Permission Denied (Ensure your cloud project has the "Smart Home Scene Linkage" or relevant lock APIs authorized).
  

#### 5. PHP Implementation Example (WordPress)

This snippet demonstrates how to authenticate, generate the v1.1 offline password, map the guest's email to the Tuya log, and save a local copy of the transaction to the WordPress database.

PHP

```
/** * Request an Offline Temporary Password from Tuya API v1.1 * * @param string $user_email      The guest's email address (used for logging & delivery) * @param int    $start_timestamp 10-digit Unix timestamp for check-in * @param int    $end_timestamp   10-digit Unix timestamp for check-out * @return string                 Success or Error message */
function issue_temporary_lock_password( $user_email, $start_timestamp, $end_timestamp ) {

    // --- ENVIRONMENT CREDENTIALS ---
    $client_id     = '<CLIENT_ID>'; 
    $client_secret = '<CLIENT_SECRET>'; 
    $device_id     = '<DEVICE_ID>'; 
    $base_url      = '<BASE_URL>'; 

    // --- TIME FORMATTING (Offline locks require hour-rounded timestamps) ---
    $tz = new DateTimeZone( 'Asia/Manila' );

    $startDT = new DateTime( '@' . $start_timestamp );
    $startDT->setTimezone( $tz );
    $startDT->setTime( (int)$startDT->format('H'), 0, 0 );
    $start_timestamp_rounded = $startDT->getTimestamp();

    $endDT = new DateTime( '@' . $end_timestamp );
    $endDT->setTimezone( $tz );
    $endDT->setTime( (int)$endDT->format('H'), 0, 0 );
    $end_timestamp_rounded = $endDT->getTimestamp();

    if ( $end_timestamp_rounded <= $start_timestamp_rounded ) {
        $end_timestamp_rounded = $start_timestamp_rounded + 3600;
        $endDT->setTimestamp( $end_timestamp_rounded );
    }

    // --- 1. GET ACCESS TOKEN ---
    $access_token = get_tuya_access_token( $client_id, $client_secret, $base_url ); 
    if ( ! $access_token ) {
        return "System Error: Authentication failed.";
    }

    // --- 2. REQUEST OFFLINE PASSWORD ---
    $offline_endpoint = "/v1.1/devices/{$device_id}/door-lock/offline-temp-password";

    $payload = array(
        'effective_time' => (int) $start_timestamp_rounded,
        'invalid_time'   => (int) $end_timestamp_rounded,
        'type'           => 'multiple', 
        'name'           => 'Guest_' . $user_email // Maps email to Tuya App Log
    );

    $create_response = tuya_api_request(
        'POST',
        $offline_endpoint,
        wp_json_encode( $payload ),
        $client_id,
        $client_secret,
        $access_token,
        $base_url
    );

    // --- 3. PROCESS RESPONSE & LOG ---
    if ( isset( $create_response['success'] ) && $create_response['success'] === true ) {

        $offline_pin = $create_response['result']['offline_temp_password'];
        $start_str   = $startDT->format( 'Y-m-d g:i A' );
        $end_str     = $endDT->format( 'Y-m-d g:i A' );

        // Save to WordPress Database History Log
        $log_entry = array(
            'email'        => $user_email,
            'pin'          => $offline_pin,
            'start_time'   => $start_str,
            'end_time'     => $end_str,
            'generated_on' => current_time( 'mysql' )
        );
        $existing_logs = get_option( 'tuya_passkey_history', array() );
        array_unshift( $existing_logs, $log_entry );
        $existing_logs = array_slice( $existing_logs, 0, 100 ); // Keep last 100
        update_option( 'tuya_passkey_history', $existing_logs );

        // Email the Guest
        $subject = 'Your Smart Lock Passkey';
        $message = "Hello!\n\nYour offline door passkey is: {$offline_pin}\n\nIt allows unlimited entries from {$start_str} to {$end_str} (Manila Time).\n\nPlease remember to press the # key after entering your PIN.\n\nEnjoy your stay!";
        wp_mail( $user_email, $subject, $message );

        return "Success! 10-Digit Offline Passkey ({$offline_pin}) generated and emailed to {$user_email}.";

    } else {
        $code = isset( $create_response['code'] ) ? $create_response['code'] : 'Unknown';
        $msg  = isset( $create_response['msg'] )  ? $create_response['msg']  : 'No message';
        return "Tuya Sync Error: {$msg} (Code: {$code})";
    }
}

/** * Helper: Generate HMAC-SHA256 Token */
function get_tuya_access_token( $client_id, $client_secret, $base_url ) {
    $endpoint       = '/v1.0/token?grant_type=1';
    $timestamp      = (string) ( time() * 1000 ); 
    $string_to_sign = "GET\n" . hash( 'sha256', '' ) . "\n\n" . $endpoint;
    $sign           = strtoupper( hash_hmac( 'sha256', $client_id . $timestamp . $string_to_sign, $client_secret ) );

    $response = wp_remote_get(
        $base_url . $endpoint,
        array(
            'headers' => array(
                'client_id'   => $client_id,
                'sign'        => $sign,
                't'           => $timestamp,
                'sign_method' => 'HMAC-SHA256',
            ),
        )
    );

    if ( is_wp_error( $response ) ) return false;
    $data = json_decode( wp_remote_retrieve_body( $response ), true );
    return ( isset( $data['success'] ) && $data['success'] === true ) ? $data['result']['access_token'] : false;
}

/** * Helper: Execute Signed API Request */
function tuya_api_request( $method, $endpoint, $body, $client_id, $client_secret, $access_token, $base_url ) {
    $timestamp      = (string) ( time() * 1000 );
    $string_to_sign = strtoupper( $method ) . "\n" . hash( 'sha256', $body ) . "\n\n" . $endpoint;
    $sign           = strtoupper( hash_hmac( 'sha256', $client_id . $access_token . $timestamp . $string_to_sign, $client_secret ) );

    $args = array(
        'method'  => strtoupper( $method ),
        'headers' => array(
            'client_id'    => $client_id,
            'access_token' => $access_token,
            'sign'         => $sign,
            't'            => $timestamp,
            'sign_method'  => 'HMAC-SHA256',
            'Content-Type' => 'application/json',
        ),
        'timeout' => 15,
    );

    if ( ! empty( $body ) ) {
        $args['body'] = $body;
    }

    $response = wp_remote_request( $base_url . $endpoint, $args );
    if ( is_wp_error( $response ) ) return array( 'success' => false, 'msg' => $response->get_error_message() );

    return json_decode( wp_remote_retrieve_body( $response ), true );
}
```
