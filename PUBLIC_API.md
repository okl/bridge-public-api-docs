#BeyondLabs public api
## 1. Using OAuth
#### Account Connecting process - needs to be done once per user
1. You'll need your Client ID `CLIENT_ID` and secret `SECRET`
2. In the following URL:
    * Replace `CLIENT_ID` in the following URL with your CLIENT_ID
    * Replace http://decorist.com/example/callback/receiver with your actual callback location
    * Send your logged-in app user to the resulting URL. `https://public-api.bbby.io/public-api/oauth/new?client_id=CLIENT_ID&redirect_uri=http://decorist.com/example/callback/receiver`
3. After the user has authorized your app, they will be directed back to your app at the redirect_uri you provided.
    * 2 more parameters will be sent, so the URL will look like this: `http://decorist.com/example/callback/receiver?code=fb214880ae40f1a57d3f96803c558bc5&response_type=code`
4. You now need to take that code and use it, along with your application secret, to get an access_token for the user. You won't have to save the code after that.
    * Make a POST request to the public api like this. Here's an example using CURL. You may use any HTTP library to do this. Make sure you correctly fill out the `client_id` and `client_secret` params with your real ones, and the `code` with the `code` we sent back with when they landed back on your site.
    ```
    curl -X POST -d '' "http://public-api.bbby.io/public-api/oauth/token?client_id=ab7257d1859b11d7241578b30ad0f311&client_secret=SECRET&code=fb214880ae40f1a57d3f96803c558bc5"
    ```
5. The public api will respond with a JSON response that includes an `access_token`!
    ```json
    {"access_token":"d0a5399eb2d0c1d60316a5bade19bd0b",
    "token_type":"bearer",
    "refresh_token":"308112d6dc5a1837e49d550301cf3ebf",
    "expires_in":false}
    ```
    You can discard the refresh_token. Also, you'll see that `expires_in` is set to false, indicating that this token does not expire.
6. Store this `access_token` and associate it to your user.

#### Using the `access_token` to make requests on behalf of your user

Now, requests can be made to any of the routes below on behalf of the user. These requests can be done in 2 ways:

##### Header Way
Include a custom header called "Authorization" - it contains the word "token", then a space, then the access_token for the user.
```
curl -H 'Authorization: token d0a5399eb2d0c1d60316a5bade19bd0b' 'http://localhost.bbby.io:3400/public-api/lists' 
```
##### URL Way
Append a URL parameter `access_token` to any request, as below. 
```
curl 'http://localhost.bbby.io:3400/public-api/lists?access_token=d0a5399eb2d0c1d60316a5bade19bd0b' 
```

#### Notes
###### Making API calls from browser vs from server
- These API calls could ultimately be made directly from the browser to the public api server, but we'll need to coordinate to allow you to make cross-domain requests. This will involve us knowing the hostname of the origin that users will be using on the decorist side (e.g. `decorist-tool.decorist.com` ). For now, the API requests would need to be made on the server side, and the results can be passed through to your user. 
###### Handling polling 
- The *Last Updated* action is the best one to use for periodic checking to see if a list has been updated since you last saw it. Just store the value that you had when you fetched the list, then if it changes you'll know you should update the list. We'll endeavour to make sure this call is as efficient as possible, but it would still be best if you detect idleness of your tab (no mouse/key input). You could then (a) refrain from polling during this period and (b) infer from a transition from Idle->Not Idle as an opportunity to poll for updates to the list, then when the user reactivates the tab, the list updates would then instantly pop in.



## 2. Endpoints
  ### Index action: `GET /public-api/lists`
  * Parameters:
    * `name`: Get all having this name (Exact match) 
    * `owner_id`: Get all having this owner
  * Returns: Object (JSON). Example:
  ```json
{
    "info": {
        "num_results": 2
    },
    "results": [
        {
            "list_id": 4,
            "owner_id": 1,
            "name": "Great List",
            "length": 3,
            "status": "active",
            "sharing": "private",
            "first_item_id": 76413360,
            "created_at": "2018-02-06T10:27:46.275Z",
            "created_by": null,
            "updated_at": "2018-02-08T20:41:45.487Z",
            "updated_by": null
        },
        {
            "list_id": 5,
            "owner_id": 1,
            "name": "Second List",
            "length": 1,
            "status": "active",
            "sharing": "private",
            "first_item_id": 76992169,
            "created_at": "2018-02-06T10:27:46.275Z",
            "created_by": null,
            "updated_at": "2018-02-06T19:07:04.759Z",
            "updated_by": null
        }
    ]
}
```

  
  ### Show action: `GET /public-api/lists/12345`
  * Parameters: List ID in path
  * Returns: Object (JSON)
    * Example:
    ```json
    {
        "list_id": 4,
        "owner_id": 1,
        "name": "Great List",
        "status": "active",
        "sharing": "private",
        "created_at": "2018-02-06T10:27:46.275Z",
        "created_by": null,
        "updated_at": "2018-02-21T00:22:32.498Z",
        "updated_by": null,
        "skus": [
                {
                  "id": 12345,
                  "sku_name": "Big Comfy Chair",
                  "images": ["http://....jpg", "http://2....jpg"],
                  "link": "http://link/..../example/12345",
                  "concept_name": "One Kings Lane or whatever",
                  "brand_name": "Brand",
                  "dimensions": "1 x 2 x 3 inches",
                  "price": 123.45,
                  "total_inventory": 0,
                  "returnable": true,
                  "margin_dollars": 111.00, // Note: may or may not be present depending on policy
                  "category": "Armchairs",
                  "color": "Blue"
                },
            {
                ...
            },
            {
                ...
            }
        ]
    }
    ```
  * Returns HTTP 404 if list not found, 403 if you are not allowed to see this list (not your list)


  ### Last updated action: `GET /public-api/lists/12345/last_updated`
  * Parameters: List ID in path
  * Returns: Object (JSON)
    * Example:
    `{":updated_at":"2018-02-21T00:29:56.919Z"}`
  * Returns HTTP 404 if list not found, 403 if you are not allowed to see this list (not your list)
