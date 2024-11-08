# Ghost SSO

**Important information** Ghost CMS does not support classic SSO so I created a custom solution.
The SSO login process will work as follows:

1.  The user clicks the "Login" button on the site.
2.  A main site (staging.escapecollective.com/sso/) opens with a modal login window.
3.  After successful authentication, the user is redirected back to the original site with a JWT token.
4.  The original site verifies the token using a public key from main site (https://staging.escapecollective.com/members/.well-known/jwks.json).
5.  Upon successful verification, the user is logged into the original site automatically.

## Step 1
You need to set up a redirect after the user clicks the "Login" button to the following URL:
`https://staging.escapecollective.com/sso/#/portal/account`

Additionally, make sure to include the `r` parameter with the URL to which the user should be redirected after a successful login.

The final URL should look like this:
`https://staging.escapecollective.com/sso/?r=https://classifieds.escapecollective.com/#/portal/account`

After a successful login the user will be redirect to `https://classifieds.escapecollective.com`

![Google Drive Image](https://drive.google.com/uc?export=view&id=1fXij7BZyKlObtvVPFTMsEbt7ltQIsy3K)

## Step 2
After registration or login on the Ghost site, the user will be redirected back to the original site (the one specified in the `r` parameter) with a `jwt` parameter in the URL.

For example:

    https://classifieds.escapecollective.com/?jwt=eyJhbGciOiJSUzUxMiIsInR5cCI6IkpXVCIsImtpZCI6IjJVTkRUUlFEa3ZQVTJjcHBBTU0zdzE4aTd5c0NLbHRyMmNwaFB3c3VVMFEifQ.eyJzdWIiOiJkLmEuZG9yb3NoY2h1a0BnbWFpbC5jb20iLCJraWQiOiIyVU5EVFJRRGt2UFUyY3BwQU1NM3cxOGk3eXNDS2x0cjJjcGhQd3N1VTBRIiwiaWF0IjoxNzMwODk2NTY4LCJleHAiOjE3MzA4OTcxNjgsImF1ZCI6Imh0dHBzOi8vc3RhZ2luZy5lc2NhcGVjb2xsZWN0aXZlLmNvbS9tZW1iZXJzL2FwaSIsImlzcyI6Imh0dHBzOi8vc3RhZ2luZy5lc2NhcGVjb2xsZWN0aXZlLmNvbS9tZW1iZXJzL2FwaSJ9.bNT35qhYrjZdcRXPnpSC_hk5hlcheyX5FHYvn5LulB6WBMVfHj7LwHoETppdPhFwIhM149CgXv1wSuePm7JSRd2zedvmOeKGUPBjD2El4XerbrDb2kSMOUGyXBQfde6aGrvrgDlhW4JNQnSR1wGCjvw0Ul8cvT9fiizVcfYkyC4

JWT looks like this:

![Google Drive Image](https://drive.google.com/uc?export=view&id=1ajSQZJ5cpueKx03K1lhoXR-yQeM13uW0)

## Step 3
Next, you need to verify the token on your end using the public key available at the following link:
`https://staging.escapecollective.com/members/.well-known/jwks.json`

    {"keys":[{"kty":"RSA","kid":"2UNDTRQDkvPU2cppAMM3w18i7ysCKltr2cphPwsuU0Q","n":"ubVDVyJ3hBAbVi8t1PdpWn0ycrVvfEKPzMAmq-1BnJ9PsLbrkV8yT8RPxgIOHidzrGwsXJhwCgC0OcUm0K6tQUizkivOy2fSnV5rYAQ2k3kxqVu9hJxva8zUDNupdVnmpUGtyILjLzZ2ASuvp2wUDVWgQm4JB_OB-d3oW4xPvfE","e":"AQAB"}]}

This can be done using different libraries, for example for PHP I have used [Firebase JWT](https://github.com/firebase/php-jwt)
For example PHP code:

    use Firebase\JWT\JWT;
    use Firebase\JWT\JWK;
    
    //$token - jwt token from GET param,
    //$jwksUrl - https://staging.escapecollective.com/members/.well-known/jwks.json
    function verify_jwt_token( $token, $jwksUrl ): array {  
      
	    $jwks = file_get_contents( $jwksUrl );  
         
	    if ( ! $jwks ) {  
	       throw new Exception( "Failed to load JWKS." );  
	    }  
         
	    $keys = json_decode( $jwks, true );  
	  
	    if ( ! isset( $keys['keys'] ) ) {  
	       throw new Exception( "Invalid JWKS format." );  
	    }  
         
	    $parsedKeys = JWK::parseKeySet( $keys, 'RS512' );  
	  
	    try {  
	       $decoded = JWT::decode( $token, $parsedKeys );  
	             
	       if ( $decoded->aud !== 'https://staging.escapecollective.com/members/api' || $decoded->iss !== 'https://staging.escapecollective.com/members/api' ) {  
	          throw new Exception( "Invalid audience or issuer." );  
	       }  
	  
	       return (array) $decoded;  
	  
	    } catch ( Exception $e ) {  
	       throw new Exception( $e->getMessage() );  
	    }
	}
## Step 4
After successfully verifying the token, you can proceed to log the user into your application. The `sub` field contains the email address of the authenticated user.

The Classifieds site is built with WordPress, which doesn’t support SSO login by default. To enable it, I used a third-party plugin, [Simple JWT Login](https://wordpress.org/plugins/simple-jwt-login/), and created a new token for authentication after verifying the JWT token from the Ghost site.

## Step 5
If you need other fields than email, you can use [Ghost Admin API](https://ghost.org/docs/admin-api/) to get subscription data and user name.
You need to make a request to the following endpoint:

`https://staging.escapecollective.com/ghost/api/admin/members/`

Make sure to use the `filter` parameter with the email you received from the JWT token (found in the `sub` field).

For example for email `d.a.doroshchuk@gmail.com`:
`https://staging.escapecollective.com/ghost/api/admin/members/?filter=email:d.a.doroshchuk@gmail.com`

Standard authorization via token (`Ghost {{admin_jwt_token}}`) is used to authorization to the API

To get this token, you need to use `admin_api_id` and `admin_api_secret` from Ghost CMS settings (I will provide them to you on request).

Generate token can be done using different libraries, for example for PHP I have used [Firebase JWT](https://github.com/firebase/php-jwt)
For example PHP code:

    use Firebase\JWT\JWT;
    use Firebase\JWT\JWK;
    
    function ghost_get_user( string $user_email ): array {  
	    $admin_api_id = '6728756b...'; // Admin API ID (part before ":")  
	    $admin_api_secret = '5316c707a...'; // Admin API Secret (part after ":")  
	    $ghost_url = '/admin/'; // Your Ghost site URL  
	    
	    // JWT Payload settings  $issued_at = time(); // Token issuance time in seconds  
	    $expiration = $issued_at + 300; // Token expiration time (5 minutes later)  
	    $payload = [  
	       'aud' => $ghost_url,  
	       'iat' => $issued_at,  
	       'exp' => $expiration  
	    ];  
  
	    // Create the JWT token  
	    $jwt_token = JWT::encode( $payload, hex2bin( $admin_api_secret ), 'HS256', $admin_api_id );  
	    
	    // API request URL  
	    $url = 'https://staging.escapecollective.com/ghost/api/admin/members/?filter=email:' . $user_email;
	    
	    // Request options with Authorization header  
	    $args = [  
	       'headers' => [  
	          'Authorization' => 'Ghost ' . $jwt_token,  
	          'Content-Type' => 'application/json'
	       ]  
       ];  
       
       // Execute the GET request  
       $response = wp_remote_get( $url, $args );  
  
	   // Check for errors  
	   if ( is_wp_error( $response ) ) {  
	       // Display error message
	       echo 'Request error: ' . $response->get_error_message();  
	       return [];  
	    } else {  
	       // Retrieve the response body and decode JSON  
	       $body = wp_remote_retrieve_body( $response );  
	       $data = json_decode( $body, true );  
  
	       $members = $data['members'];  
  
	       if ( $members ) {  
	          return (array) $members[0];  
	      }  
    }

The result of this query, if all is true, will return the user's data by email, for example:

    {  
      "id": "671f96a06788bcdfeb66bf6c",  
      "uuid": "7bb62ade-08a8-4d22-867d-a1e0210e9eef",  
      "email": "d.a.doroshchuk@gmail.com",  
      "name": "Denis Dor.",  
      "note": null,  
      "geolocation": "{\"country\":\"Slovakia\",\"region\":\"Bratislava Region\",\"latitude\":\"48.1833\",\"longitude\":\"17.0379\",\"accuracy\":100,\"ip\":\"172.225.104.162\",\"timezone\":\"Europe/Bratislava\",\"organization\":\"AS36183 AKAMAI-AS\",\"country_code\":\"SK\",\"asn\":36183,\"area_code\":\"0\",\"organization_name\":\"AKAMAI-AS\",\"city\":\"Bratislava\",\"country_code3\":\"SVK\",\"continent_code\":\"EU\"}",  
      "subscribed": false,  
      "created_at": "2024-10-28T13:50:24.000Z",  
      "updated_at": "2024-11-06T07:14:44.000Z",  
      "labels": [  
        {  
          "id": "671f5cb56eac97d6855b8eb3",  
          "name": "Former_Comp_Subscriber",  
          "slug": "former_comp_subscriber",  
          "created_at": "2024-10-28T09:43:17.000Z",  
          "updated_at": "2024-10-28T09:43:17.000Z"  
	      }  
      ],  
      "subscriptions": [  
        {  
          "id": "",  
          "tier": {  
            "id": "671f45f16eac97d6855b8e87",  
            "name": "Membership - GBP",  
            "slug": "membership-gbp",  
            "active": true,  
            "welcome_page_url": null,  
            "visibility": "public",  
            "trial_days": 0,  
            "description": "Get unlimited access to our content and become part of our community",  
            "type": "paid",  
            "currency": "GBP",  
            "monthly_price": 799,  
            "yearly_price": 8900,  
            "created_at": "2024-10-28T08:06:09.000Z",  
            "updated_at": "2024-10-29T15:13:41.000Z",  
            "monthly_price_id": null,  
            "yearly_price_id": null,  
            "expiry_at": null  
	      },  
	          "customer": {  
	            "id": "",  
	            "name": "Denis Dor.",  
	            "email": "d.a.doroshchuk@gmail.com"  
	      },  
          "plan": {  
            "id": "",  
            "nickname": "Complimentary",  
            "interval": "year",  
            "currency": "USD",  
            "amount": 0  
	      },  
          "status": "active",  
          "start_date": "2024-10-29T15:13:39.000Z",  
          "default_payment_card_last4": "****",  
          "cancel_at_period_end": false,  
          "cancellation_reason": null,  
          "current_period_end": null,  
          "price": {  
            "id": "",  
            "price_id": "",  
            "nickname": "Complimentary",  
            "amount": 0,  
            "interval": "year",  
            "type": "recurring",  
            "currency": "USD",  
            "tier": {  
              "id": "",  
              "tier_id": "671f45f16eac97d6855b8e87"  
	      }  
          },  
          "offer": null  
      }  
      ],  
      "avatar_image": "https://www.gravatar.com/avatar/921dfd3bec46b40b23595d2ad52f5f3b?s=250&r=g&d=blank",  
      "comped": true,  
      "email_count": 0,  
      "email_opened_count": 0,  
      "email_open_rate": null,  
      "status": "comped",  
      "last_seen_at": "2024-11-06T07:14:43.000Z",  
      "email_suppression": {  
        "suppressed": false,  
        "info": null  
      },  
      "newsletters": []  
    }
