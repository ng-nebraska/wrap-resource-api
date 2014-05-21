## Extending Angular's $resource for Consistent REST API's

One of my favorite modules provided by AngularJS is the [**ngResource**](https://docs.angularjs.org/api/ngResource/service/$resource) module,
because it has intelligent defaults and also allows me to be very flexible configuring
resources and REST api calls to match pretty much any non-standard url structure
that can arise from legacy services.

This flexibility can come with a cost when you need to have many REST endpoints that
do not match up with the standard patterns that **$resource** expects.

For example: If you need to add an HTTP PUT method for an `update()` function on your
resources, each resource configuration would need to look similar to below:


```javascript
var Timesheet = $resource('/timesheets/:id',
  {
    id: '@id'
  },
  {
    update: {
      method: 'PUT'
    }
  }
);

```

- This registration automatically gives you:
  - a POST to `/timesheets` with a JSON request body via `data.create()`
  - a GET to `/timesheets` with parameters via `data.list()`
  - a GET to `/timesheets/:id` via `data.get()`
  - a PUT to `/timesheets/:id` with a JSON request body via `data.update()`
  - a DELETE to `/timesheets/:id` via `data.remove()`

But if ***EVERY*** domain model in your application needs the update method and has an `id`
property, there would be a ton of duplication littered throughout your controllers and/or
services because you would need to declare your resources wherever they are used.

For this reason, I like to wrap my `$resource` objects with a 2 step approach: *Registration*
and *Execution*. I accomplish this by creating two simple services. We'll call them `api` and `data`.

> Here is a gist of the entire code we'll review in this post: https://gist.github.com/brucecoddington/92a8d4b92478573d0f42

&nbsp;

## Resource Registration: api Service

The `api` service handles the registration of all of our applications' resources and provides
us a way to set default configuration for properties that are very specific to our
applications' needs. Creating these resources is what the `api` service is supposed
to make easy and seamless.

> For demonstration purposes, we will assume all of our services require an id and update function.

Let's step through the code to see what it does:

 - First, we create an Angular module and set the `ngResource` module as a dependency.

```javascript
angular.module('app.resources', ['ngResource'])
```

- Our `api` service is a factory with the `$resource` service injected as a dependency.

```javascript
.factory('api', function ($resource) {
```

- The first property of our `api` is a configuration object that contains the mapping of
the id parameter to the `id` property of the resource object. Any other cross-cutting
url variables or query parameters can be placed within this object and named appropriately.

```javascript
    var api = {
      defaultConfig : {id: '@id'},
```
- We then set an `extraMethods` object on our service to contain any extra method configuration
that is needed by our application but not provided by default via `$resource`.

```javascript
      extraMethods: {
        'update' : {
          method: 'PUT'
        }
      },
```
- Our `api` service will have one method, `add()`. This method will handle registering
each resource with the service and make them available to the rest of the application.
It also lets us configure our resources to be specific to our needs.

```javascript
      add : function (config) {
        var params,
          url;
```
- We first check to see if the `config` parameter is a string. If it is, we create a
configuration based upon its value.
 - The resource key is the value of the string parameter.
 - The url is just the value with a forward slash prepended to it.


- The resulting configuration is then assigned to the function's config object.  

```javascript
        // If the add() function is called with a
        // String, create the default configuration.
        if (angular.isString(config)) {
          var configObj = {
            resource: config,
            url: '/' + config
          };

          config = configObj;a
        }
```
- If the configuration's `unnatural` property has not been set to *tru-ish*, we can
set up the objects to be used to create each new **$resource**.  
 - We first copy the service's default configuration and extend it with the `params`
 object in the function's `config` parameter.
 - Second, we automatically append the `id` path variable to the config's url.

```javascript
        // If the url follows the expected pattern, we can set cool defaults
        if (!config.unnatural) {
          var orig = angular.copy(api.defaultConfig);
          params = angular.extend(orig, config.params);
          url = config.url + '/:id';
```

- If the `unnatural` property has been set to `true`, we require that the params and url
be explicitly set. This gives us the freedom to completely control each **$resource**'s
configuration when we have to use an endpoint that doesn't particularly follow our expected
pattern.  

```javascript
        // otherwise we have to declare the entire configuration.
        } else {
          params = config.params;
          url = config.url;
        }
```

- If we pass in a method configuration, we will use it as our resource's methods parameter, but
the default is to use the `extraMethods` object that we set up previously.

```javascript
        // If we supply a method configuration, use that instead of the default extra.
        var methods = config.methods || api.extraMethods;
```

- Once our configuration is completed, we can register the new resource object as a
property on our `api` service so that we can access it within our application.

- We also return the api instance so that we can chain the registrations and remove
even more duplication.

```javascript
        api[config.resource] = $resource(url, params, methods);
        return api;
      }
    };
```

- At the end of our factory recipe, we return the `api` object that we just created.

```javascript
    return api;
  });
```

- With this method, we can now inject our `api` service as a dependency and register our
resource models with an easy one-line call:

```javascript
api.add('timesheet');
```
- which is exactly the same as:

```javascript
api.add({
  resource: 'timesheet',
  url: '/timesheet'
});
```
- both of which translate to the initial resource configuration at the beginning of this post.

###### But what if we need a special configuration?
If the `timesheets` service url also contains a user's id that owns the timesheet, we
can still use the `api` service to configure for that special case:

```javascript
api.add({
  resource: 'timesheets',
  url: '/users/:user_id/timesheets',
  params: {
    user_id: '@user_id'
  }
});
```

##### This gives the ability to make all of the http calls listed above on our resources.
For example, if we want to get the timesheet with the id of 1234, we would have 2 options:

```javascript
api.timesheet.get({id: 1234}).$promise.then(function () {}); // object property style

api['timesheet'].get({id: 1234}).$promise.then(function() {}); // array syntax style
```

> This still gives us quite a bit of duplication with all of the awkward $promise's needed
> to access each resource's promise to use the $q services easier to read syntax.

##### That is where the Execution step, or the `data` service, provides us with a more friendly api.

&nbsp;


## Execution Step: data Service

Now that we have the ability to register resources with our `api` service, we can create
a service to wrap each api call with an easy to follow api and return the promises that
each method creates when called.

- We will use a provider so that we can use its methods from within **$httpProvider** or
**$stateProvider** configuration's `resolve` block in our module's `config()` when
we are declaring routes or states. More on this later..

```javascript
  .provider('data', {
```

- First we inject our newly created `api` service into our provider's `$get` function so
that it's resources are available.


- Each function within our `data` service takes 2 parameters:
  - The name of the resource
  - The query object or model that is used to set the path variables and query parameters
  in each of our HTTP calls.


- Each function also returns the promises created by each HTTP method call so that we
can use them to attach success and error resolutions.  

```javascript
    $get: function (api) {

      var data = {

        list : function (resource, query) {
          return api[resource].query(query).$promise;
        },

        get : function (resource, query) {
          return api[resource].get(query).$promise;
        },

        create : function (resource, model) {
          return new api[resource](model).$save().$promise;
        },

        update : function (resource, model) {
          return api[resource].update(model).$promise;
        },

        remove : function (resource, model) {
          return api[resource].remove(model).$promise;
        }
      };

      return data;
    },
```
- So once our `data` service is injected into a controller, we can use it to make RESTful
calls to our services to retrieve or edit our application's data.

```javascript
// Get the list of timesheets based upon a query object
data.list('timesheets', query)
  .then(function (timesheets) {
    $scope.timesheets = timesheets;
  });

// Call the static $update method on an existing resource.
$scope.timesheet.$update()
  .then(function (updated) {
    $scope.timesheet = updated;
  })
  .catch(function (x) {
    // handle the error
  });

// Create a new timesheet
data.create('timesheets', timesheet)
  .then(function (created) {
    // notify the user..
  })
  .catch(function (x) {
    // handle the error...
  });
```

### The Power of Providers

I mentioned earlier that there was a reason we selected to use a ***Provider*** for our
`data` service, instead of just a ***Factory*** or ***Service*** recipe. That is mainly
because of the power of the `resolve` block in the ***$routeProvider*** and/or ***$stateProvider***
services.

Since the configuration of routes/states happens within a module's config block, one way
to make it easy to put data queries in a resolve block is to use methods on a provider
to proxy to the service's methods at runtime.

For example, setting `list()` and `get()` functions on our ***dataProvider*** ...

```javascript
  list: function (resource, query) {
    return [
      'data',
      function list (data) { // inject the data service
        data.list(resource, query);
      }
    ]
  },

  get: function (resource, query) {
    return [
      'data',
      function get (data) { // inject the data service
        data.list(resource, query);
      }
    ]
  }
```
... allows us to easily use these functions in our resolve blocks, like:

```javascript
resolve: {
  timesheets: dataProvider.list('timesheets', $stateParams), // get the list or..
  timesheet : dataProvider.get('timesheets', $stateParams)   // get the individual
}
```

Hopefully these pointers help you develop a clean api for your application model's data
querying and manipulation. This is just one method that the teams I've worked
with have gravitated to make it a little more simple to manage the code duplication in
declaring and configuring resources and also make it easy to normalize how we
worked with promises when we talked to our REST services.

How are you doing it? I would love to see other examples, especially if it would make
my life easier in the long run. I would greatly appreciate any comments on a better
method. Thanks for taking the time to read this post. Cheers!
