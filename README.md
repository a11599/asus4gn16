**Reverse engineering the ASUS 4G-N16 router's admin API**

The ASUS 4G-N16 is a relatively cheap CAT 4 LTE router with detachable 2x2 MIMO LTE antennas, built-in wifi and one ethernet port. It has band locking and cell locking capabilities. I use it in an area where I had to use band lock during a given timeframe only (since the cell covering that band seems to be shut down during the night). I solved this by reverse engineering and simulating the actions performed on the admin panel with a PHP script executed from a Cron job on a Raspberry Pi Zero 2W.

In this document I share my findings on the router's admin API and provide example PHP code using the php_curl extension. The examples are kept minimal on purpose. You can turn this into a class or something without globals for a cleaner code if you wish.

The router's admin panel is a [Single-page application](https://en.wikipedia.org/wiki/Single-page_application) that communicates with the router's API endpoints. It makes a surprisingly large amount of requests for something as simple as a router admin panel IMO. This however makes it a lot easier to simulate admin panel actions because there is no need to rely on the UI of the admin panel (which also means better chance to survive firmware updates - not that there are many new updates for this router).

# API endpoints

Admin functions seem to be implemented on two endpoints:

- `/reqproc/proc_get` handles all HTTP GET requests;
- `/reqproc/proc_post` handles all HTTP POST requests.

Both endpoints require a `User-Agent` HTTP header. Not sure what kind of validation is performed by the router. I used `User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36` which worked fine. The router will abort the request in the absence of this header.

Both endpoints respond with JSON data, but the response is sometimes not a valid JSON (it may contain multiple values with the same object key).

Requests that are protected from public access require the `random` session cookie to contain the ID of the authenticated session.

In PHP you need to obtain a CURL handle first. The two curl_setopt commands instruct CURL to use the in-memory cookie storage and to start with no cookies (these are not strictly necessary, but don't hurt either). Adjust `$routerBaseURL` to point to the base address of the router's admin panel.

```php
$routerBaseURL = 'http://192.168.0.1';
$ch = curl_open();
curl_setopt($ch, CURLOPT_COOKIEFILE, '');
curl_setopt($ch, CURLOPT_COOKIELIST, 'ALL');
```

## GET requests

GET requests expect parameters in the URL [query string](https://en.wikipedia.org/wiki/Query_string).

Required parameters are:

- `cmd=<command>`: Defines the data to be retrieved from the router.
- `isTest=false`: A constant that just needs to be passed as-is.

Inspect network communication in the browser to find out which commands are supported. Some GET requests are available without authentication, for example `system_status`, `web_signal`, `network_type` and `ppp_status` seem to return at least partial info even without authentication. It seems that multiple commands can be executed with one request by using a comma-separated list of commands. The admin panel seems to also use the `multi_data=1` URL parameter in such cases.

To send a GET request in PHP:

```php
function router_get($command) {
    global $ch, $routerBaseURL;

    curl_setopt($ch, CURLOPT_URL, "$routerBaseURL/reqproc/proc_get?cmd=$command&isTest=false");
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36'
    ]);
    curl_setopt($ch, CURLOPT_POST, 0);

    return json_decode(curl_exec($ch), true);
}
```

The function returns the response of the router in an associative array.

## POST requests

POST requests expect parameters in the request body in `application/x-www-form-urlencoded` MIME type format. This is basically the format of a query string as if they were a GET request and as such they need to be URL encoded (PHP makes this easy by using the `http_build_query()` function).

Required parameters are:

- `goformId=<command>`: Defines the command/action to be performed.
- `isTest=false`: A constant that just needs to be passed as-is.

Inspect network communication in the browser to find out which commands are supported. Most POST requests require an authenticated session. These requests also require a CSRF token in the `CSRFToken` parameter.

To obtain a CSRF token perform a GET request with the `get_token` command. A random number will be returned in the `token` property of the response JSON. To use it as a CSRF token, hash it with SHA-256 algorithm. If the session is not yet authenticated, the request won't fail, but the response will not contain the `token` property.

To obtain a CSRF token in PHP:

```php
function router_getCSRFToken() {
    $data = router_get('get_token');
    $token = '';

    if ($data && array_key_exists('token', $data)) {
        $token = hash('sha256', $data['token']);
    }

    return $token;
}
```

The returned value must be passed to the POST request as its `CSRFToken` parameter, immediately following this request. If the session is not authenticated, the function returns an empty string.

To send a POST request in PHP:

```php
function router_post($command, $parameters) {
    global $ch, $routerBaseURL;

    $token = router_getCSRFToken();
    if ($token) {
        $parameters['CSRFToken'] = $token;
    }
    $parameters['isTest'] = 'false';
    $parameters['goformId'] = $command;

    curl_setopt($ch, CURLOPT_URL, "$routerBaseURL/reqproc/proc_post");
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36'
    ]);
    curl_setopt($ch, CURLOPT_POST, 1);
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($parameters));

    return json_decode(curl_exec($ch), true);
}
```

The function returns the response of the router in an associative array. This will work for both authenticated and unauthenticated sessions (the admin panel UI also seems to always call `get_token` before every POST request).

# Authentication

You need to start an authenticated session to access most API commands. The process of authentication is as follows:

- Get a password salt by doing a GET request with `GET_RANDOM_LOGIN` command. The command returns a random value in the `random_login` JSON property.
- Create an SHA-256 hash by appending the `admin` user's password to the password salt.
- Perform a POST request with the `LOGIN` command. Set the `username` parameter to the base64 encoded value of `admin` (`YWRtaW4=`) and the `password` parameter to the base64 encoded value of the password hash created in the previous step.
- The request shall return a value less than, or equal to `1` in the `result` JSON property when the authentication was successful.
- It will also return a `Set-Cookie` header with a session ID-like value for the `random` cookie.

To authenticate in PHP:

```php
function router_login($password) {
    $data = router_post('GET_RANDOM_LOGIN', []);
    if (!$data || !array_key_exists('random_login', $data)) {
        return false;
    }
    $salt = $data['random_login'];
    $data = router_post('LOGIN', [
        'username' => 'YWRtaW4=',
        'password' => base64_encode(hash('sha256', $salt . $adminPassword))
    ]);

    if (!$data || $data['result'] > 1) {
        return false;
    }

    return true;
}
```

The function returns `true` if the authentication was successful or `false` otherwise.

## Logging out

To close the authenticated session, perform a POST request with the `LOGOUT` command.

To log out in PHP:

```php
function router_logout() {
    $data = router_post('LOGOUT', []);

    if ($data && $data['result'] == 'success') {
        return true;
    }

    return false;
}
```

The function returns `true` if the logoff was successful or `false` otherwise.

# Performing router operations

The building blocks above are enough to perform any operation programmatically. Inspect network communication in the browser while clicking through the admin panel to find out what is executed by a GUI operation.

For example in my case I had to disable band 20 during a specific timeframe to obtain higher download speeds and better response times. I checked what is sent to the API after clicking apply on the band locking tab. The admin panel sent the `TZ_SET_LOCK_BAND` command in the `goformId` parameter with the following additional parameters:

- `band_state` = `yes`
- `band_list` = `69,0,0,0,160,0,0,0` when band 20 was disabled, `69,0,8,0,160,0,0,0` when all bands were enabled
- `wcdma_list` = `129.0.0` (does not matter in my area since 3G is decommissioned)
- `tds_list` = `0.0`
- `zeact` = `2`
- Plus the usual `isTest` and `CSRFToken` parameters

Band locking can now be changed programmatically with the following PHP code:

```php
function controlBand20($disableBand20, $routerAdminPassword) {
    $bandList = ($disableBand20 ? '69,0,0,0,160,0,0,0' : '69,0,8,0,160,0,0,0');

    if (!router_login($routerAdminPassword)) {
        echo "Router login failed!\n";
        return;
    }

    $data = router_post('TZ_SET_LOCK_BAND', [
        'band_state' => 'yes',
        'band_list' => $bandList,
        'wcdma_list' => '129.0.0',
        'tds_list' => '0.0',
        'zeact' => '2'
    ]);
    if ($data && $data['result'] != 0) {
        echo "Band lock command failed!\n";
        return;
    }

    router_logout();
}
```

Some operations like cell locking require a reboot of the router. It is best to avoid these operations, but if you really need to do this, make sure to wait for an adequate time before trying to access the router again (measure reboot time and add some extra safety margin).
