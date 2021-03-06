# Restify ACL


[![Build Status](https://travis-ci.org/ojengwa/restify-acl.svg?branch=master)](https://travis-ci.org/ojengwa/restify-acl)
[![Coverage Status](https://coveralls.io/repos/github/ojengwa/restify-acl/badge.svg?branch=master)](https://coveralls.io/github/ojengwa/restify-acl?branch=master)
[![Codacy Badge](https://api.codacy.com/project/badge/Grade/43a996be6b1a4153aac74751115d97a0)](https://www.codacy.com/app/bernard/restify-acl)


Restify Access Control Lists (restify-acl) enables you to manage the requests made to your restify server. It makes use of ACL rules to protect your sever from unauthorized access. ACLs defines which user groups are granted access and the type of access they have against a specified resource. When a request is received against a resource, `restify-acl` checks the corresponding ACL policy to verify if the requester has the necessary access permissions.

##### What are ACL rules
ACL is a set of rules that tell `restify-acl` how to handle the requests made to your server against a specific resource. Think of them like road signs or traffic lights that control how your traffic flows in your app. ACL rules are defined in JSON or yaml syntax.

**Example**
``` json
[{
  "group": "user",
  "permissions": [{
    "resource": "users",
    "methods": [
      "POST",
      "GET",
      "PUT"
    ],
    "action": "allow"
  }]
}]

```
YAML syntax

```yaml

- group: user
  permissions:
    - resource: users
      methods:
        - GET
        - POST
        - DELETE
      action: allow

```

The contents of this file will be discussed in the usage section


## Installation

You can download `restify-acl` from NPM
```bash

$ npm install restify-acl

```

then in your project require restify-acl

``` js

const acl =  require('restify-acl');

```

 or GitHub

 ```
$ git clone https://github.com/ojengwa/restify-acl.git

  ```
copy the lib folder to your project and then require `nacl.js`

``` js

const acl =  require('./lib/nacl');

```

# Usage

Express acl uses the configuration approach to define access levels.

## Configuration 

First step is to create a file called `nacl.json` and place this in the root folder. This is the file where we will define the roles that can access our application, and the policies that restrict or give access to certain resources. Take a look at the example below.

```json

[{
  "group": "admin",
  "permissions": [{
    "resource": "*",
    "methods": "*"
  }],
  "action": "allow"
  }, {
  "group": "user",
  "permissions": [{
    "resource": "users",
    "methods": [
      "POST",
      "GET",
      "PUT"
    ],
    "action": "deny"
  }]
}]

```

In the example above we have defined an ACL with two policies with two roles,  `user` and `admin`. A valid ACL should be an Array of objects(policies). The properties of the policies are explained below.

Property | Type | Description
|--- | --- | ---|
  **group** | `string` | This property defines the access group to which a user can belong to  e.g `user`, `guest`, `admin`, `trainer`. This may vary depending with the architecture of your application.
  **permissions** | `Array` | This property contains an array of objects that define the resources exposed to a group and the methods allowed/denied
|**resource** | `string` | This is the resource that we are either giving access to. e.g `blogs` for route `/api/blogs`, `users` for route `/api/users`. You can also specify a glob `*` for all resource/routes in your application(recommended for admin users only)
  **methods** | `string or Array` | This are http methods that a user is allowed or denied from executing. `["POST", "GET", "PUT"]`. use glob `*` if you want to include all http methods.
  **action** | `string` | This property tell restify-acl what action to perform on the permission given. Using the above example, the user policy specifies a deny action, meaning all traffic on route `/api/users` for methods `GET, PUT, POST` are denied, but the rest allowed. And for the admin, all traffic for all resource is allowed.

#### How to write ACL rules
ACLs define the way requests will be handled by restify acl, therefore its important to ensure that they are well designed to maximise efficiency. For more details follow this [link](https://github.com/ojengwa/restify-acl/wiki/How-to-write-effective-ACL-rules)

## Authentication
restify-acl depends on the role of each authenticated user to pick the corresponding ACL policy for each defined user groups. Therefore, You should always place the acl middleware after the authenticate middleware. Example using jsonwebtoken middleware

``` js
  // jsonwebtoken powered middleware
  server.use(function(req, res, next) {
   const token = req.headers['x-access-token'];
    if (token) {
      jwt.verify(token, key, function(err, decoded) {
        if (err) {
          return res.send(err);
        } else {
          req.decoded = decoded;
          next();
        }
      });
    }
  });

  // restify-acl middleware depends on the the role
  // the role can either be in req.decoded (jsonwebtoken)or req.session
  // (restify-session)

  server.use(acl.authorize);

```

# API
There are two API methods for restify-acl.

### config[type: function, params: config { filename<string>,path<string>, yml<boolean>, encoding, baseUrl, rules}, response {}]

This methods loads the configuration json file. When this method it looks for `nacl.json` the root folder if path is not specified.

### config
- **filename**: Name of the ACL rule file e.g nacl.json
- **path**: Location of the ACL rule file
- **yml**: when set to true means use yaml parser else JSON parser
- **baseUrl**: The base url of your API e.g /developer/v1
- **rules**: The rules you can direct set for nacl

```js
  const acl = require('restify-acl');

  // path not specified
  // looks for config.json in the root folder
  // if your backend routes have base url prefix e.g  /api/<resource>,  v1/<resource> ,
  // developer/v1/<resource>
  // specify it in the config property baserUrl {baseurl: 'api'} ,
  // {baseurl: 'v1'}, {baseurl: 'developer/v1'} respectively
  // else you can specify {baseurl: '/'} or ignore it entirely


  acl.config({
    baseUrl:'api/v1'
  });

  // path specified
  // looks for ac.json in the config folder

  acl.config({
    filename:'acl.json',
    path:'config'
  });

  // When specifying path you can also rename the json file e.g
  // The above file can be acl.json or nacl.json or any_file_name.json

  acl.config({
	rules: rulesArray
  });

  // When you use rules api, nacl will **not** to find the json/yaml file, so you can save your acl-rules with any Database;

```
### response
This is the custom error you would like returned when a user is denied access to a resource. This error will be bound to status code of `403`

```js

const acl = require('restify-acl');

let configObject = {
    filename:'acl.json',
    path:'config'
};

let responseObject = {
  status: 'Access Denied',
  message: 'You are not authorized to access this resource'
};

acl.config(configObject, responseObject);

```

## authorize [type: middleware]
This is the middleware that manages your application requests based on the role and acl rules.

```js

app.get(acl.authorize);

```
## unless[type:function, params: function or object]
By default any route that has no defined policy against it is blocked, this means you cannot access this route untill you specify a policy. This method enables you to exclude unprotected routes. This method uses express-unless package to achive this functionality. For more details on its usage follow this link [express-unless](https://github.com/jfromaniello/express-unless/blob/master/README.md)

```js
//assuming we want to hide /auth/google from restify acl

app.use(acl.authorize.unless({path:['/auth/google']}));

```

Anytime that this route is visited, unless method will exlude it from being passed though our middleware.

**N/B** You don't have to install `express-unless` it has already been included into the project.

# Example
Install restify-acl

```
$ npm install restify-acl

```

Create `nacl.json` in your root folder
```json
[{
  "group": "user",
  "permissions": [{
    "resource": "users",
    "methods": [
      "POST",
      "GET",
      "PUT"
    ],
  "action": "allow"
  }]
}]

```

Require restify-acl in your project router file.

```js
const acl = require('restify-acl');
```

Call the config method

```js
acl.config({
  //specify your own baseUrl
  baseUrl:'/'
});

```

Add the acl middleware

```js
app.use(acl.authorize);
```

For more details checkout the [examples folder](https://github.com/ojengwa/restify-acl/tree/master/examples).

# Contributions
Pull requests are welcome. If you are adding a new feature or fixing an as-yet-untested use case, please consider writing unit tests to cover your change(s). For more information visit the contributions [page](https://github.com/ojengwa/restify-acl/wiki/contributions)
