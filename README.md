# Using Vue.js with headless Drupal
Most guides on decoupling Drupal only seem to cover simple `GET` requests. But what about other operations like `POST` or `DELETE`?
This Vue.js template includes ready to use methods to make CRUD operations using OAuth2

# Configure & Build Vue.js Project
1. Rename the `.env.local.example` to `.env.local` file in the folder.

3. Set `VUE_APP_API_URL` as your Drupal installation URL.

4. Set `VUE_APP_CLIENT_ID` as `UUID` value from step 4 in Drupal / Authentication & Rest section.

5. Set `VUE_APP_CLIENT_SECRET` as `New Secret` value from step 4 in Drupal / Installation & configuration section.

6. If needed, rename `vue.config.js.example` to `vue.config.js` and make the appropriate changes.

7. Run `npm run serve`



# Configure Drupal CMS

## Installation & configuration
This guide assumes that there is already a Drupal project created and setup.

We will use OAuth 2.0 as an authentication method using a Simple OAuth module and built-in modules for REST. 
There are detailed installation instructions on modules official page. 

Here are the simplified steps:

1. Install the module using the following command and enable it.

```sh
composer require drupal/simple_oauth drupal/restui
```

2. Generate keys using the following command, in this example, we will store them one folder up from where Drupal is installed.

```sh
openssl genrsa -out private.key 2048
openssl rsa -in private.key -pubout > public.key
```

3. Save the path to your keys in: `/admin/config/people/simple_oauth`. Using the location from Step 2, then the paths should be `../public.key` and `../private.key`. Also setting token expiration time from 300 to 86400 is a good idea.

4. Create a Client Application by going to: `/admin/config/services/consumer/add`. 
Remember what you've set in `New Secret` field value and `UUID` value, which is shown after saving. 
These values will be used in the Vue .env.local

5. To make REST endpoints (POST, PATCH, DELETE) work, go to `/admin/config/services/rest` and enable **Content** resource. 
After enabling it, click edit. Select all methods, select `json` as accepted request format and `oauth2` as the authentication method.

After this, Drupal should be ready to go.



## CORS
This is an important issue I've encountered. If Vue app and Drupal REST endpoints are hosted in different domains, you need to enable CORS in Drupal. Here is how you do it:

1. Copy `sites/default/default.services.yml` to `sites/default/services.yml`.
2. Edit the `sites/default/services.yml` to look like this: 
```
parameters:
  cors.config:
        enabled: true
        allowedHeaders: ['x-csrf-token','authorization','content-type','accept','origin','x-requested-with', 'access-control-allow-origin','x-allowed-header']
        allowedMethods: ['POST', 'GET', 'OPTIONS', 'PATCH', 'DELETE']
        allowedOrigins: ['*']
        exposedHeaders: true
        maxAge: false
        supportsCredentials: true
```

3. Flush all caches.


## REST Authentication
Since we are using OAuth 2.0 as our authentication method, the only REST endpoint for auth we will use is `/oauth/token`. This endpoint can be used to login and to refresh the token.


#### Login
To use `/oauth/token` endpoint for login you must do `POST` request, with headers `Content-Type: application/x-www-form-urlencoded` and request body must contain following keys:

- `grant_type` with value `password`
- `username` and `password`, which user that will be logging it.
- `client_id` and `client_secret`, which are values from step 4 in Drupal / Installation & configuration section.

This will return `access_token` and `refresh_token`. `access_token` should be used for all further HTTP request as `Bearer {access_token}`.

#### Refresh
To use `/oauth/token` endpoint for a token refresh you must do `POST` request, same as for Login, but two changes:

- No `username` and `password` in the request body.
- Add `refresh_token` stored from the Login request in the request body.


## REST API
If you visit `/admin/config/services/rest` you can see which paths you can use as endpoints. Two requirements:

- All the requests need authentication, which is `Bearer {access_token}` in the request header.
- All requests must have `?_format=json` in the end, so it returns JSON.

 Apart from that Drupal categorizes all requests as safe (GET) and unsafe (POST, PATCH, DELETE, PUT):


#### Safe requests
You can perform these requests without any problem. A couple of examples:

- `/node/{node}` - If you make a GET request, you will get node details.
- `/{view}` - Only GET request can be made to this path because it is a custom endpoint generated by view. 


#### Unsafe requests
These requests must have `X-CSRF-Token: {csrf_token}` in the request header. An example:

- `/node/{node}` - You can make a POST request to create a node, PATCH request to update node and DELETE request to remove a node.


#### CSRF Token
You can get CSRF Token by performing GET request to `/session/token`.

