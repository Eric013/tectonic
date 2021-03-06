# Manager API

## Instantiation API

```
const manager = new Manager({
  // the resolver matches queries against your source definitions (API endpoints)
  // and invokes them.  This is pluggable so you can write your own, but 99% of
  // the time you want to use the built in Resolver.
  //
  // As of v1.3 this has no options.
  resolver: new Resolver(),

  // drivers specifies all of the drivers for use when data loading.  Drivers
  // communicate with API endpoints and return data given a query.
  //
  // Each driver can have source definitions specified using `manager.driver.X`,
  // where X is the object key name.
  //
  // For example, to specify source definitions which use the superagent driver
  // using the below code we can do the following:
  //
  //   // each source definition specified here uses the superagent driver
  //   // for communication
  //   manager.fromSuperagent([...])
  //
  drivers: {
    // use the Superagent driver under `manager.fromSuperagent
    fromSuperagent: new TectonicSuperagent(),
    fromMock: new Mock(),
  },

  // store specifies your redux store using the tectonic reducer as the key
  // `tectonic`
  store: store,
})
```

### Drivers

**Driver setup**

To supply source definitions you need to set up manager using one or more
drivers.  Once you have the manager instance, all drivers are available
under `manager.drivers.X`, where X is the key of the driver specified in
setup.

For example:

```
const manager = new Manager({
  // ...
  drivers: {
    fromSuperagent: new TectonicSuperagent(),
    foo: new FooDriver(),
  },
});

// We can define sources which use the "foo" driver via:
manager.drivers.foo([...]);

// And these sources will use the "superagent" driver:
manager.drivers.fromSuperagent([...]);
```

This is so you can use many drivers per project; in some apps you might need
a standard AJAX driver plus a WebSocket driver to subscribe to a feed (note:
actual subscriptions need to be implemented. For now, we need to poll which
is still easy).

## Source definition API

Once you've set up drivers you can begin adding your data sources to tectonic.
Below is an example containing the full API and comments on each option for a
source definition.

These source definitions are used when a query is passed in to tectonic to
load/update/delete data.  Each query has attributes (eg. 'GET' a single User
model) which is matched against all source definitions in order to perform
a request.

For more information on how queries are matched to source definitions see
INTERNALS-RESOLVER.md

```
// Import the models that you've defined within your project. For more
// information on defining models see the model API guide.
import UserModel from './models/user.js';
import PostModel from './models/post.js';

// Define a manager with some drivers
const manager = new Manager({
  // ...prior setup
  drivers: {
    fromSuperagent: new TectonicSuperagent(),
    foo: new FooDriver(),
  },
});

// Define your sources
manager.drivers.fromSuperagent([
  {
    // 'meta' is driver-specific implementation data. The below `meta` object
    // is for the superagent driver only.
    meta: {
      // Specify URL endpoint to hit. All ':params' will be replaced with
      // params supplied in the query.
      url: '/api/v1/users/:id',

      method: 'GET',

      // If necessary the superagent driver allows you to transform data
      // before being sent to tectonic.
      // Note that as this is within `meta` this is driver specific.
      transform: (data) => {
        return data;
      },
    },

    // **Common parameters**
    // The below parameters are given for every source definition, regardless
    // of the driver.

    // 'id' is an **optional** unique ID for this source which can be helpful
    // during debugging.  This is randomly assigned when left undefined.
    id: 'foo',

    // 'params' specifies **required parameters** for this source.  If
    // undefined this source has no required parameters.
    //
    // This **must** always be an array of strings.
    //
    // When a source definition has required params **only queries with these
    // params will be considered**.
    //
    // In this example queries without the 'id' param will never use this
    // source definition:
    //
    //  // Won't match; the query from `.getItem` has no params
    //  @load({
    //    user: User.getItem(),
    //  })
    //
    //  // Will match; the query from `.getItem` has an 'id' param
    //  @load({
    //    user: User.getItem({ id: '1' }),
    //  })
    //
    // For more information on creating queries see API-DECORATOR.md
    params: ['id'],

    // 'optionalParams' specifies optional params for a source.  These do not
    // need to be specified in a query to use this source.
    optionalParams: ['foo'],

    // 'returns' specifies what data this source definition returns. Some API
    // endpoints return a **single item**, whilst others return a
    // **list of items**.
    //
    // What's more, some API endpoints return more than one model.  You can
    // specify all of these in this key:
    //
    // > Returns a single model item (ie. the API response is an object ofk
    //   model data):
    //     returns: UserModel.item(),
    //
    // > Returns a list of models (ie. the API response is an array);
    //     returns: UserModel.list(),
    //
    // > Returns many models:
    //
    //     // this returns an object containing a user as an object and their
    //     // posts as an array.
    //     returns: {
    //       user: UserModel.item(),
    //       posts: PostModel.list(),
    //     }
    //
    //   Note that when returning many models the API response must match the
    //   object specified in `returns`.
    //
    //   In this example the endpoint must return an object with the keys
    //   "user" and "posts"; "user" must be an object containing all of the
    //   user data and "posts" must be an array of objects containing
    //   post data.
    //
    //   The superagent driver can transform data before it is passed to the
    //   resolver to be validated.  This allows you to make sure that the
    //   response fulfils the `returns` contract without having to change
    //   your API response. For more information see the DRIVER-SUPERAGENT
    //   docs.
    //
    // Finally, some API queries return no data (eg. a DELETE request may
    // return a "204 No Content"). For these queries returns may be omitted
    // but a "model" key must be specified.
    returns: UserModel.item(),

    // 'model' must **only** be specified for sources which return no data
    // and have no 'returns' key (eg. DELETE queries which delete a particular
    // model but return no data).
    //
    // The resolver needs a way to determine which model a source acts on,
    // and whether the model returns data.
    //
    // This should always be the base model class.
    model: UserModel,

    // 'queryType' is a string specifying the operation being performed. Valid
    // values are:
    //
    //  - 'GET' (default when 'queryType' is undefined), for sources with no
    //    side effects that return data.
    //  - 'CREATE', for sources which create new resources
    //  - 'UPDATE', for sources which update resources (eg. PUT/PATCH) requests
    //  - 'DELETE', for sources which delete resources
    //
    // Each query passed into the resolver also has a queryType.  The type for
    // the query and source definition must match to process a query.
    //
    // This may be omitted; the default is 'GET'.
    queryType: 'GET'
  },
  // this API endpoint creates a new user
  {
    // 'meta' holds driver-specific data
    meta: {
      url: '/api/v1/users',
      method: 'POST',
    },
    queryType: 'CREATE',
    returns: UserModel.item(),
  },
  // this API endpoint deletes a user
  {
    // 'meta' holds driver-specific data
    meta: {
      url: '/api/v1/users',
      method: 'DELETE',
    },
    queryType: 'DELETE',
    model: UserModel,
  },
]);
```

