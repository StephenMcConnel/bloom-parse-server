# bloom-parse-server

This is the database backend for bloom-library.org, using the [parse-server](https://github.com/ParsePlatform/parse-server) module.

Here is the full [Parse Server guide](https://github.com/ParsePlatform/parse-server/wiki/Parse-Server-Guide).

### Set Up For Local Development

1. Make sure you have at least Node 4.3.

    `node --version`
1. Clone this repo and go into its directory, and install or update the dependencies:

    `npm install`

1. Install mongodb server

1. Give mongodb a blank directory to work with, and run it:

    `mongod.exe --dbpath c:\temp\mongodata`

1. Start up this server:

    `npm start`

    Or, to debug, open bloom-parse-server in vscode, F5 (Debug: Launch via NPM). Note that this sets the masterid to "123", via an environment variable.

1. Setup or update the mongodb Schema

    ```
    curl -X POST -H "X-Parse-Application-Id: myAppId" -H "X-Parse-Master-Key: 123" -d "{}" http://localhost:1337/parse/functions/setupTables
    ```
    You should get

    `{"result":"SetupTables ran to completion."}`

    and see the tables in the dashboard.

1. Run Parse Dashboard:

    Go to [http://localhost:1337/dashboard](http://localhost:1337/dashboard)

    You will be required to log in. The username is the appId ('myAppId'), and the password is the masterKey ('123').

### Dashboard

See above for setting up the dashboard locally.

Public dashboards:

* Production: [http://parsedashboard.bloomlibrary.org](http://parsedashboard.bloomlibrary.org)
* Development: [http://dev-parsedashboard.bloomlibrary.org](http://dev-parsedashboard.bloomlibrary.org)

You will be required to log in. The username is the appId, and the password is the masterKey.

### Sample Queries

```
curl -X POST \
  -H "X-Parse-Application-Id: myAppId" \
  -H "Content-Type: application/json" \
  -d '{}' \
  http://localhost:1337/parse/functions/hello
```

### Deployment

Notes below on Azure Setup are relevant to deployment, but I wanted to separate out the exact steps a developer would go through to deploy changes.

#### develop branch

Once changes have been pushed to the develop branch,
1. Go to the Azure portal (portal.azure.com). Access must be granted by LTOps.
2. Open the bloom-parse-server-develop app service.
3. Open Deployment slots.
   - Note that steps 2 and 3 can be skipped by opening the staging app service directly.
4. Open bloom-parse-server-develop-staging.
5. Open "Deployment Center" for the staging app service.
6. Wait until your changes have been sucessfully deployed (check the status column).
7. Repeat steps 2 and 3.
8. Click Swap.
9. Review settings changes to make sure no app service settings are getting changed accidentally.
10. Click Swap.
11. Deployment and restart of the service can take several minutes.
    - During this time, the dashboard and library part of the website will be down.

#### master branch

Once changes have been merged to the master branch,
1. Follow the same steps as develop except the app service names are
    - bloom-parse-server-production
    - bloom-parse-server-production-staging
    
#### modifying the schema

Once the changes have been deployed, you can run the setupTables function in main.js to modify the schema. See notes there.

### Azure Setup

We are running three different services:

* bloom-parse-server-unittest
* bloom-parse-server-develop
* bloom-parse-server-production

Each is backed by a single mongodb at mlab.com. This is how they were made:

1. Create the mongodb on mlab.com, making sure to select Azure and the same datacenter. Failing to do this increases response times by 3x.
2. In Azure, create a new "Web App" App Service
3. In Azure:App Service:Application Settings:App Settings, create these settings:

    DATABASE_URI mongodb://account:password@something.com:port/database

    APP_ID you make this up.

    MASTER_KEY you make this up

    SERVER_URL http://[azure app service name].azurewebsites.net/parse

    Note: Don't leave off that /parse in the SERVER_URL!

    REST API: parse.com used this (not clear what the actual environment variable would be) but as far as I can tell, the open source Parse-Server does not.

4. In the App Service's Deployment settings, add a slot for staging and point that staging app service at this github repository,
with the appropriate branch. A few minutes later, parse-server will be running on the staging app service.
Note that Azure apparently does the `npm install` automatically, as needed.
The staging app service automatically redeploys when github notifies it of a checkin on the branch it is watching.
The staging app service can then be swapped out with the live one.
See the deployment section above for detailed steps.

5. We never touch the schema using the Parse Dashboard or letting queries automagically add clases or fields.
Instead, we set up the schema using a Cloud Code function `setupTables`.
If you haven't set up the database already, follow instructions shown above under "Setup or update the mongodb Schema".
Use Azure:App Service:Log stream to monitor progress.
Note: During one setup, I found this can be flaky, perhaps becuase I jumped the gun.
So instead I did the curl post for `functions/testDB`, which worked.
Then I tried `functions/setupTables` again, and this time it worked.

6. TODO: Backup, Logging setup.
