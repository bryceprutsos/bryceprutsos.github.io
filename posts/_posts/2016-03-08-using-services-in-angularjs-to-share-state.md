---
layout: post
title: Using services in AngularJS to share state
description: >
  This post walks through using an AngularJS service to share state
redirect_from:
    - /post/using-services-in-angularjs-to-share-state
---

In AngularJS services can be used to share state across multiple views and controllers. This can be useful for several reasons. It can allow the application to go between views without having to wait on web requests to go get the data again from the server.  Another useful benefit is to allow 2 controllers to share the same data. For example if you add an item in an editing controller you may want to see that change reflected in a summary chart at the same time.

As an example imagine an application that allows a user to design a donut with the toppings and dough flavor they select. To maintain the state of the user’s selections an AngularJS service will be used. To identify which donut the user is creating a donutId will be stored in the $state AngularJS service. The application also has a service that provides access the web API for the donut application.

The first thing to do is create the service and inject the dependencies:

```js
(function (DonutApplication) {
    'use strict';
    DonutApplication.Services.factory('donutStateService', ['$state', '$q', 'donutApiService',
        function ($state, $q, donutApiService) {
            
         }]);
})(DonutApplication);
```

The service is named “donutStateService” and has dependencies on the $state and $q AngularJS services and the donutApiService.

The first thing to do in the service is define the model that will contain the state:

```js
var vm = {
    toppings: [],
    dough: null
};
```

It is important to always return this model from the service and not the individual contents of it. The reason is that an object must be shared in order for changes to be reflected. If a variable is shared then the controller will only get a copy of the value rather than a reference to it.

The next step is to create variables to represent the API resources needed and the current donutId loaded into the service’s model:

```js
var toppingsResource = donutApiService.toppingsResource();
var doughResource = donutApiService.doughResource();

var curDonutId = 0;
```

Add methods to populate the model from the API:

```js
function getToppings() {
    var defer = $q.defer();
    toppingsResource.query({ donutId: curDonutId }, {}, 
        function (data) {
            vm.toppings = data;
            defer.resolve(data);
        });
    return defer.promise;
}

function getDough() {
    var defer = $q.defer();
    doughResource.query({ donutId: curDonutId }, {}, 
        function (data){
            vm.dough = data;
            defer.resolve(data);
        });
    return defer.promise;
}
```

Create a method to check the $state’s donutId against the donutId loaded into the service’s model. If they do not match then the Id should be updated and the model should be refreshed from the API. The items in the model are initialized individually because resetting the vm would cause any consumers’ references to be broken.

```js
function resetIfDonutChanged() {
    var defer = $q.defer();
    if (curDonutId !== $state.params["donutId"]) {
        curDonutId = $state.params["donutId"];
        vm.toppings = [];
        vm.dough = null;
        
        $q.all(
            getToppings(),
            getDough()
        )
        .then(function () {
           defer.resolve(); 
        });
    } else {
        defer.resolve();
    }
    return defer.promise;
}
```

The caller of the donutStateService should get a promise back to allow the state service to update the model from the API if necessary. In order to simplify this a helper function is useful:

```js
function returnPromisedValues(values) {
    var defer = $q.defer();
    resetIfDonutChanged().then(function () {
        defer.resolve(values);
    });
    return defer.promise;
}
```

Finally create the return object of the donutStateService and return the model wrapped in a promise:

```js
return {
    getDonutInfo: function () {
        return returnPromisedValues(vm);
    }
};
```

To use the donutStateService in a controller you could set a variable on the scope to the object returned by the getDonutInfo method:

```js
$scope.vm.donutDetails = donutStateService.getDonutInfo();
```

By accessing the state service in this way across multiple controllers changes made in one controller will be reflected in the other. This is a simple, but useful pattern to speed up and simplify an AngularJS application that needs to share state across multiple views.
