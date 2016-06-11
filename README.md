roseredis
=========

Easy to use abstraction layer for node_redis

## Installation
  ```
  npm install roseredis --save
  ```

## Usage Example

  ```javascript
  var rose = require('roseredis');
  var roseClient = rose.createClient();

  var redisKey = {
    foo: 'fooKey',
    bar: 'barKey'
  };

  var commands = {
    setFoo:
    function(value) {
      return ['set',redisKey.foo,value];
    },
    getFoo:
    function() {
      return {
        command: ['get',redisKey.foo],
        handler: function(reply, result) {
          return result.foo = reply;
        }
      };
    },
    setBar:
    function(value) {
      return ['set',redisKey.bar,value];
    },
    getBar:
    function() {
      return {
        command: ['get',redisKey.bar],
        handler: function(reply, result) {
          result.bar = parseInt(reply) || 0;
        }
      };
    },
  };

  roseClient.registerCommands(commands);

  roseClient.multi()
    .setFoo('Foo is set to this')
    .setBar(123456)
    .getFoo()
    .getBar()
    .exec(function(err, result) {
      console.log(result.foo);
      // Foo is set to this
      console.log(result.bar, typeof result.bar);
      // 123456 'number'
    });

  ```
## Creating the client

A rose client can be created by calling rose.createClient()
```javascript
var rose = require('roseredis');
var roseClient = rose.createClient();
```

If you already have a redis client configured, you may pass this in.
```javascript
var rose = require('roseredis');

var redis = require('redis');
var redisClient = redis.createClient();

// Configure your redisClient here

var roseClient = rose.createClient(redisClient);
```

Once the rose client has been created, you may access it's redisClient at anytime from roseClient.redisClient
```javascript
roseClient.redisClient.quit();
// This will disconnect from the redis server

```
  
You may create child clients that inherit the parent's scope at the time of creation.  
Commands registered in the child client will not affect the scope of the parent client.  
Commands registered to the parent after the child has been created will not affect the child's scope.
```javascript

roseClient.registerCommand('commandA', function(){});

var childClient = roseClient.createClient();

childClient.registerCommand('commandB', function(){});

roseClient.registerCommand('commandC', function(){});

console.log( typeof roseClient.commandA ); // function
console.log( typeof roseClient.commandB ); // undefined
console.log( typeof roseClient.commandC ); // function

console.log( typeof childClient.commandA ); // function
console.log( typeof childClient.commandB ); // function
console.log( typeof roseClient.commandC ); // undefined

```

Full documentation of node_redis can be found here:  
https://github.com/NodeRedis/node_redis#readme

## Creating commands

Rose commands are methods that return a redis command array and optional reply handler.

```javascript
function getFoo() {
  return {
    command: ['get', 'fooKey'],
    handler: function(reply, result) {
      result.foo = reply;
    }
  };
}
```
Command is the redis command array.  
(This gets the value stored at redis key 'fooKey')
```javascript
command: ['get', 'fooKey'],
```
Handler wraps the reply from redis.  
reply is the redis response from your command  
result is the object that gets returned in the callback  
  
This allows you to give your result values meaningful names, instead of having to keep track of reply indices
```javascript
handler: function(reply, result) {
  result.foo = reply;
}
```

You can optionally return just the redis command if you're not doing anything with the redis reply.  
(This is common for setters)
```javascript
function setFoo(value) {
  return ['set', 'fooKey', value];
}
```

You can use the handler to provide additional formatting of the data, such as parsing values from the reply.
```javascript
handler: function(reply, result) {
  result.bar = parseInt(reply) || 0;
}
```


## Registering commands
Once created, you'll need to register commands with the rose client.  
  
This can be done one at a time, using client.registerCommand(label, method);  
```javascript
function setFoo(value) {
  return ['set', 'fooKey', value];
}

roseClient.registerCommand('setFoo', setFoo);
```

Or, more typically, you can use registerCommands to register all the commands defined in an object
```javascript
var commands = {
  setFoo:
  function(value) {
    return ['set',redisKey.foo,value];
  },
  getFoo:
  function() {
    return {
      command: ['get',redisKey.foo],
      handler: function(reply, result) {
        result.foo = reply;
      }
    };
  }
};
roseClient.registerCommands(commands);
```

Use result.setKey to set a deeply nested field in the result object.
```javascript
function getFooBar() {
  return {
    command: ['hget','foo', 'bar'],
    handler: function(reply, result) {
      result.setKey('foo.bar', reply);
    }
  };
}

roseClient.registerCommand('getFooBar', getFooBar);
roseClient.getFooBar(function(err, result) {
  // result.foo.bar will be set to the reply value
});
```
(See the deepref github page for full documentation of setKey)  
https://github.com/isaymatato/deepref  

## Executing commands
Once registered, you may execute a single command by calling roseClient[commandName]  
```javascript
function setFoo(value) {
  return ['set','foo',value];
}

roseClient.registerCommand('setFoo', setFoo);
roseClient.setFoo(42, function() {
  // redis key 'foo' will now be set to '42'
});
```
  
You may also execute multiple commands using roseClient.multi()  
```javascript
roseClient.multi()
  .setFoo('Foo is set to this')
  .setBar(123456)
  .getFoo()
  .getBar()
  .exec(function(err, result) {
    console.log(result.foo);
    // Foo is set to this
    console.log(result.bar, typeof result.bar);
    // 123456 'number'
  });
  
  ```
## Tests
  ```
  npm test
  ```

## Contributing

Please use the included style guide.  If you change anything, please test
and add unit tests for any new functionality.  If you're fixing a bug, please
add a unit test that would have failed before the bug was fixed.  Thanks!