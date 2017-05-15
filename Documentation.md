# Application flow/implementation:
* The application is implemented using Spring Boot framework.
* All the supported end points are implemented in shim-server/common/controller
  * LegacyAuthorizationController - Authorization related logic
  * LegacyDataPointSearchController - Data fetching related logic.
  * {New addition} Notification controller - Supports notifications from vendors.
* Based on the end point request from the client, The corresponding controller is executed, where it processes the request and forwards to the requested 'shim' (refer to shim-server/shim). These are vendor-specific implementations and any logic that is limited to a particular vendor should go here. 
  * In the case of data, a typical shim implementation would do -
     1) Identify the end point and other necessary parameters required to make a call.
     2) Fetch the keys and other authentication information from the database.
     3) Make the vendor API call and get the response.
     4) If the request asked for data normalization, pass the response to the respective "DataPointMapper". They contain specific implementations to convert the response JSON into conforming shimmer-schemas. 
     5) The resulting response is passed to the client as a JSON response.
   * For authorization requests, shimmer would identify the shim, produce url to present to the user (where he has to authenticate you to pull their data) and set the redirect url (configured in the .yaml file). Once the vendor responds with keys, they are stored in the database under the supplied 'userid'. For all the future requests, it would simply retrieve the keys and make the request.

To summarize, shimmer would handle both authorization and fetching data points. They also handle normalization of data conforming to shimmer-schema.

## Supporting a new Vendor:
* Refer to the vendor's API documentation page and understand how the API is supported (endpoints, steps to register, security features etc.)
* Add a new entry in 'application.yaml' file so that the registry can pick it up.
* Create a new package under the vendor's name.
* Understand the various interfaces available in shimmer. There are different interfaces for each mode of authentication (Oauth1, Oauth2 etc.) ShimBase is the base interface for supporting all endpoints a shim needs to.
* Start implementing them in your classes and look out for additional parameters the vendor might need (user-id etc).
* If you want to conform the responses to shim-schemas, you need to implement datapoint mappers separately for each type of data.

However, core shimmer does not support data storage and notifications offered by some vendors. Notifications, in general is a way for vendors to send updates as and when a user syncs his device. To support notifications, we need a callback url. We need a way to handle them both.

## New features

* Notification support - using the 'notify' endpoint, you can subscribe to notifications. (Currently, only withings has the implementation. Extend it for other vendors).
* New endpoints and mappers - Added missing endpoints and mappers that are needed for our application.
* Data storage - shimmer already needs a mongodb instance to run. It is possible to extend the feature to support data storage as well. It is not feasible to continously poll for new data especially when the application has to scale for thousands of users. A better way would be to listen for notifications, and do a full sweep of users who synced and store their data into mongodb. That brings us to schedulers.
* Scheduling - Once a user is subscribed to get notifications, we store them in the 'DataSync' collection which maintains the state information per user. Periodically, we query the data to fetch users having new data and perform API requests. To avoid head of the line blocking due to API limits, scheduling happens asynchronously and each vendor is assigned with a new thread.
* API Rate Limiter - Blocks the execution if the specified API rates are reached. YOu need to customize this call for each vendor.






# Notes
* Refer to the readme file on adding developer keys for different vendors.
* install-natively.sh won't work for windows (even with git shell). You need to manually run the commands from the script.
* The 'data' endpoint is not secure. It is strongly recommended that the endpoint is closed or authentication is implemented.
* Modify the ScheduledTasks.java to suit your scheduling needs (or add configuration to handle it.)
  
