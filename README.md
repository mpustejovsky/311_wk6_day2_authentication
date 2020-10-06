# Express Authentication

PLEASE NOTE THAT PRIOR TO THIS ASSIGNMENT, YOU NEED TO HAVE FINISHED THE SECTION TITLED "Authentication with Auth0" IN THE PRE-HOMEWORK

## Setup

1. Initialize and run the app: `npm install` to watch the dependencies be downloaded

2. Create a `.env` file with the following structure. Alter the fields wrapped in `<  >` to reflect your Google Cloud SQL database setup. These will be the same credentials we used to set up a connection in MySQL Workbench.

The other values can be found on your Auth0 profile after completing the pre-homework.

```yaml
    DB_HOST=<mySQLWorkbench_Connect_Hostname>
    DB_USER=root
    DB_PASSWORD=<mySQLWorkbench_Connect_Password>
    DB_DEFAULT_SCHEMA=<mySQLWorkbench_Connect_DefaultSchema> || admin
    AUTH0_IDENTITY=my-express-app
    AUTH0_DOMAIN=<findOnAuth0-APIs>>my-express-app>>test>>Node.js>>https://EVERYTHINGbtwHEREandHERE/oauth/token>
    AUTH0_CLIENT_ID=<sameAsAbove_butFind"client_id"inTheBody>
    AUTH0_CLIENT_SECRET=<sameAsAbove_butFind"client_secret"inTheBody>
```

> *NOTE: Don't include quotes in the `.env` file. If you're having trouble creating a connection with your DB in Google Cloud, try inputting the values directly into the `connection.js` file.*

4. Navigate to the `sql/connections.js` file and confirm you under stand why the fields are using `process.env.SOMETHING`. Where are they coming from?

```js
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_DEFAULT_SCHEMA
```

> *NOTE: Line 1 in `sql/connections.js` is invoking a method on the `dotenv` package. What do you think that is doing?

5. The app is using `nodemon` so you can use `npm start` and any changes made (and saved) will cause the server to restart automatically.

6. Finally, in MySQL Workbench, run the `initialize.sql` script that is included in this project. You will run this on the "admin" database. You can simply copy the sql from the file into your MySQL Workbench console. Follow the steps under the `imgs` folder if you are having trouble with this.

## Overview

Note: This is a tough project, but hang in there and try to understand as much as possible. Auth0 is a popular framework for authentication. A lot of the setup is done for us.

The `routes/controllers`, SQL statements and basic setup has been done for us. Our job is now to complete the functions in the middleware folder and then use them in our routes.

Keep in mind that your port (4001) may be different.

## GET users

In Postman or a web browser, navigate to `http://localhost:4001/users/` (a GET request) and you should be able to see a list of all the users in your database. This is information that we will leave public.

## POST / PUT / DELETE

These routes are for manipulating the data and these are things that we ideally want someone to be logged in for before they are able to work with the data. To start, we will create a middleware function (that Auth0) provides us. We also want to create an empty `.env` file in the main folder. We will use this to hold all of the Auth0 environment variables for our application. We don't want to upload these to Github.

### Middleware

In the `middleware/index.js` file, locate the function called `checkJwt`. We will need to make some modifications to this function before it will work properly.

1. Notice the `AUTH0_IDENTITY` variable. We need to find the actual identifier you created in your Auth0 account. If you followed the pre-homework it will be called `my-express-app`. If you didn't do this during the Setup section then create an environment variable in the `.env` file. The file should look like this:

```yaml
AUTH0_IDENTITY=my-express-app
```

2. Notice the `AUTH0_DOMAIN` variable. This is the domain associated with your account. This one is a little harder to find. Essentially... it's your your tenant id (from Auth0) followed by `.us.auth0.com`. You can find the tenant ID to the left of your profile in the upper right-hand corner of the Auth0 page (when signed in). So for example if your tenant id is "dev-t4vriwms" then your domain will be "dev-t4vriwms.us.auth0.com". You likely did this in the setup steps but if now, make sure to add this to the `.env` file as well. It will now look like this:

```yaml
AUTH0_IDENTITY=my-express-app
AUTH0_DOMAIN=dev-t4vriwms.us.auth0.com
```

Now we need to apply this middleware to the routes you want to protect. Before you do that though... go to Postman and send a POST request to `http://localhost:4001/users/` with no body. This should add a pre-selected user to your DB. You should have gotten a response that looks like this:

```json
{
    "newId": 501
}
```

In order to prevent this, we need to go to that route, the third one down in the `routers/users.js` file, and add `checkJWt` in between the path and the request/response function. The final result should look like this:

```js
// routes/users line 10
router.post('/', checkJwt, usersController.createUser)
```

Now go ahead and attempt to make that POST request again in Postman. The one to `http://localhost:4001/users/`. Try it a couple of times. You should now get a response with a 401 status code and a body that has `UnauthorizedError: No authorization token was found` in it. That's good news, we are now blocking requests to this endpoint until people are authenticated. Add that same middleware to the rest of the routes (that are not GET requests) in the `routers/users.js` file.

## Authenticating

So how do users authenticate? We've seen how we can block them but now we actually need certain users to have access. So we need to send the bearer token (jwt). First we'll do a test run.

### Testing the JWT

From the Auth0 dashboard in your browser, navigate to APIs (on the left side) -> My Express App (the API you created in pre-homework) -> Test. Midway down the page in the code block labeled "Response", copy the "access token" to your clipboard. It should start with `eyJ0eXAiOiJKV1QiLCJh...` or some close combination of characters.

Now in Postman, we're going to execute that same POST request but this time we are going to add a header. A header as we know is a piece of information that gets sent along with the request. In Postman go to the "headers" column and give it the name "Authorization" and the value of your token preceded by the word "Bearer". So it will look something like:

`Authorization: Bearer eyJ0eXAiOiJKV1QiLCJh...`

Execute the request and notice that you are allowed to add users again and see a response that looks like this:

```json
{
    "newId": 502
}
```

### Obtaining the JWT

Ok so we now have protected routes and some users can access them if they have the appropriate token but where do they get that token from? We need to create a workflow that sends back a token when a user logs in. We need to do that by calling an Auth0 endpoint during the login endpoint.

Find the "login" function in [controllers/auth.js](./controllers/auth.js). You'll see that the call the the Auth0 endpoint is mostly complete but we still need to do a few things.

1. Set the default directory on your Auth0 account to "Username-Password-Authentication". You can do this by clicking on your profile icon in the top right corner of your dashboard and selecting "Settings". On the settings page scroll down to "API Authorization Settings" -> "Default Directory".

2. There are two other environment variables we need to set in our `.env` file. They are "AUTH0_CLIENT_ID" and "AUTH0_CLIENT_SECRET". You can find this information in the same place we copied the test bearer token from. In the first box find the "client_id" and "client_secret" keys. Again, if you haven't already, ddd them to your `.env` file. The complete file should now look like this:

```yaml
AUTH0_IDENTITY=my-express-app
AUTH0_DOMAIN=dev-t4vriwms.auth0.com
AUTH0_CLIENT_ID=60nkmegUxOFGMn...
AUTH0_CLIENT_SECRET=BkAX6wMgD5OhRGRnFYRNBgueTX...
```

NOTE: You may need to change and un-change something in your code in order for nodemon to register these new environment variables. Saving the files does that for us.

3. Now we need to create a couple of users for our application to use. We will do this manually for now. Go to the main page of your Auth0 dashboard in your browser, select "Users & Roles" -> "User", then click "Create User". Leave the default connection type and enter an email and password for your user. For example:

```yaml
email: test@example.com
password: Password!

test1@email.com
testPassword1
```

Remember this information because we are about to use it again. This time go to Postman and submit a request to the `/auth/login` route that's already been created for us in this application. The full request will be a POST to `http://localhost:4001/auth/login` with the body:

```json
{
    "username": "test@example.com",
    "password": "Password!"
}
```

If everything worked correctly you should have received an "access_token" in return. That's the token that we can use to send to your endpoints after a user logs in. Keep in mind that the "users" are now stored in Auth0 and are different from the users we have in our database. Those database users are just dummy data at this point. We won't need to actually store information about our users because Auth0 will do it for us.

If you are having issues you may try going to Auth0 >> Settings/Dashboard >> Applications >> my-express-app >> Show Advanced Settings >> Grant Types >> And selecting "Password"

## BONUS - logger

Create a function called `logger` in the `middleware/index.js` file. It's purpose will be to log the route and date/time that each request happened. The outline of the function will look like this:

```js
const logger = (req, res, next) => {
  
}
```

Inside of this function we will put a `console.log` statement with three arguments separated by a comma:

1. The string, `'Logging route:'`
2. The request path ex. `/users`
3. The date/time in ISO format. Ex. `new Date().toISOString()`

Remember to call the `next()` function in order to continue. Otherwise, the API call will get hung up in this middleware function.

Import this logger function into the main `index.js` file: `const { logger } = require('./middleware')`

Between the `bodyParser` and the users router add the following: `app.use(logger)`

This is an example of application specific middleware. Every route will now pass through our logger function and log the path and the date/time that the request was made. This would be useful for determining our most popular routes.

## Summary

If all went according to plan we now have an API that is locked down with authentication and we have also added middleware on all of our routes that logs the current request and the associated date/time.
