# Bridge public api
## 1. Using OAuth2
This API uses the well-known OAuth2 method of authenticating users securely without needing to share the user's password between apps. It's implemented in a number of readily available third-party libraries for pretty much any language or framework, but it's also very simple and easy to build your own client.
 
#### Account Connecting process - needs to be done once per user
1. You'll need your Client ID `CLIENT_ID` and secret `SECRET`
2. In the following URL:
    * Replace `CLIENT_ID` in the following URL with your CLIENT_ID
    * Optionally (but recommended), send a `STATE` in the URL with a pseudo-random value which will be used later to determine if the caller is trusted.  This may not be needed for server-to-server calls where we know the IP address(es) of the server.
    * Optionally (but recommended), send a `SCOPE` of `read` since you are requesting read access to lists for now.  We may need additional scopes down the line for other access.
    * Replace http://decorist.com/example/callback/receiver with your actual callback location
    * Send your logged-in app user to the resulting URL. `https://public-api.bbby.io/public-api/oauth/new?client_id=CLIENT_ID&state=STATE&scope=SCOPE&redirect_uri=http://decorist.com/example/callback/receiver`
3. After the user has authorized your app, they will be directed back to your app at the redirect_uri you provided.
    * 3 more parameters will be sent, so the URL will look like this: `http://decorist.com/example/callback/receiver?code=fb214880ae40f1a57d3f96803c558bc5&response_type=code&state=STATE`
    * NOTE: We have not implemented the sending of the STATE parameter in the callback as of this writing, but we will do so to make sure there is a closed loop without a man in the middle.  This would require the caller to maintain this state on the user to make sure that it matched.
4. You now need to take that `code` and use it, along with your application secret, to get an `access_token` for the user. You won't have to save the code after that.
    * Make a POST request to the public api like this. Here's an example using CURL. You may use any HTTP library to do this. Make sure you correctly fill out the `client_id` and `client_secret` params with your real ones, and the `code` with the `code` we sent back with when they landed back on your site.
    ```
    curl -X POST -d '' "http://public-api.bbby.io/public-api/oauth/token?client_id=ab7257d1859b11d7241578b30ad0f311&client_secret=SECRET&code=fb214880ae40f1a57d3f96803c558bc5&state=STATE&&redirect_uri=http://decorist.com/example/callback/receiver"
    ```
    * Passing the redirect in again allows us to protect from XSS because we will match against the first one sent to make sure they are identical.
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

##### Header Way (DESIRED)
Include a custom header called "Authorization" - it contains the word "Bearer", then a space, then the access_token for the user.
```
curl -H 'Authorization: Bearer d0a5399eb2d0c1d60316a5bade19bd0b' 'http://localhost.bbby.io:3400/public-api/lists' 
```
##### URL Way (Might be helpful for testing, but should probably not be used from the browser in production)
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
                  	"brand_name": "Portfolio No.6",
                  	"category_name": "Ornaments",
                  	"color": "multi",
                  	"color_family": "white",
                  	"concept_name": "One Kings Lane",
                  	"dimensions": "2.3\" L x 2\" W x 4.25\" H",
                  	"primary_image": "https://okl.scene7.com/is/image/OKL/vmf_vendor_QXJ_4705522_1503961657559_734661",
                  	"pdp_url": "https://www.onekingslane.com/search.do?query=77396214",
                  	"margin_amount": "31.08",
                  	"retail_price": "89.0",
                  	"returnable": false,
                  	"sku_id": 77396214,
                  	"sku_name": "Teardrop Glass Ornaments, S/3",
                  	"total_avail_qty": 1,
                  	"alternate_images": [
                  		"https://okl.scene7.com/is/image/OKL/vmf_vendor_QXJ_4705522_1503961940441_947302"
                  	]
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