Events and the Event Dispatcher
===============================

Objected Oriented code can come a long way in ensuring extensibility
of your projects. By creating classes that have well defined
responsibilities, you make your code more flexible.

If a user of your class want to modify its behavior, he can extend
the class. But if the user want to share its changes with other
users that also have their own changes, inheritance is moot. That's
the case for instance when you want to provide a plugin system for
your class. A plugin should be able to add methods, or do something
before or after a method is executed, without interfering with other
plugins work. That's tougher to resolve.

Enter Symfony Event Dispatcher. The library implements the
[Observer](http://en.wikipedia.org/wiki/Observer_pattern) pattern in
a simple and effective way to make all these things possible and
make your projects truly extensible (see the recipes section below
for some possible implementation of these patterns).

The main goal of Symfony Event Dispatcher is to allow objects to
communicate together without knowing each other. It is possible
thanks to a central object, the *dispatcher*.

Objects (*listeners*) can *connect* to the dispatcher to listen to
specific events, and some others can *notify* an *event* to the
dispatcher. Whenever an event is notified, the dispatcher will call
the listeners.

Events
------

Unlike many other observer implementations, you don't need to create
a class to create a new event. All events are of course still
objects, but all events are instances of the built-in `sfEvent`
class.

>**NOTE**
>You can of course extends the `sfEvent` class to specialize an event
>further, or enforce some constraints, but most of the time it would add
>a new level of complexity that is not necessary.

An event is uniquely identified by a string. By convention, it is
better to use lowercase letters, numbers and underscores (`_`) for
event names. Furthermore, to better organize your events, a good
convention is to prefix the event name with a namespace followed by
a dot (`.`).

Here are examples of good event names:

    [php]
    change_culture
    response.filter_content

As you might have noticed, event names contain a verb to indicate
that it relates to something that happens.

The Dispatcher
--------------

The dispatcher is the object responsible for connecting the
listeners and calling them whenever an event is notified.

By default, the dispatcher class is `sfEventDispatcher`:

    [php]
    $dispatcher = new sfEventDispatcher();

Event Objects
-------------

The event object, of class `sfEvent`, stores information about the
notified event. Its constructor takes three arguments:

  * The *subject* of the event (most of the time, this is the object notifying
    the event, but it can also be `null`);

  * The event name;

  * An array of parameters to pass to the listeners (an empty array
    by default).

As most of the time an event is called from an object context, the
first argument is almost always `$this`:

    [php]
    $event = new sfEvent($this, 'user.change_culture', array('culture' => $culture));

The event object has several methods to get event information:

  * `getName()`: Returns the identifier of the event.

  * `getSubject()`: Gets the subject object attached to the event;

  * `getParameters()`: Returns the event parameters.

The event object can also be accessed as an array to get its
parameters:

    [php]
    echo $event['culture'];

Connecting Listeners
--------------------

Obviously, you need to connect some listeners to the dispatcher to
make it useful. A call to the dispatcher `connect()` method
associates a PHP callable to an event.

The `connect()` method takes two arguments:

  * The event name;

  * A PHP callable to call when the event is notified.

>**NOTE**
>A [PHP callable](http://www.php.net/manual/en/function.is-callable.php)
>is a PHP variable that can be used by the `call_user_func()` function
>and returns `true` when passed to the `is_callable()` function. A
>string represents a function, and an array can represent an object
>method or a class method.

    [php]
    $dispatcher->connect('user.change_culture', $callable);

Once a listener is registered in the event dispatcher, it waits
until the event is notified. The event dispatcher keeps a record of
all event listeners, and knows which ones to call when an event is
notified.

>**NOTE**
>The listeners are called by the event dispatcher in the same order you
>connected them.

For the example above, `$callable` will be called by the dispatcher
whenever the `user.change_culture` event is notified by an object.

When calling the listeners, the dispatcher passes them an `sfEvent`
object as parameter. So, the listener receives the event object as
its first argument.

Notifying Events
----------------

Events can be notified by using three methods:

 * `notify()`
 * `notifyUntil()`
 * `filter()`

### `notify`

The `notify()` method notifies all listeners in turn.

    [php]
    $dispatcher->notify($event);

By using the `notify()` method, you make sure that all the listeners
registered on the notified event are executed but none can return a
value to the subject.

### `notifyUntil`

In some cases, you need to allow a listener to stop the event and
prevent further listeners from being notified about it. In this
case, you should use `notifyUntil()` instead of `notify()`. The
dispatcher will then execute all listeners until one returns `true`,
and then stop the event notification:

    [php]
    $dispatcher->notifyUntil($event);

The listener that stops the chain may also call the
`setReturnValue()` method to return back some value to the subject.

The notifier can check if a listener has processed the event by
calling the `isProcessed()` method:

    [php]
    if ($event->isProcessed())
    {
      $ret = $event->getReturnValue();

      // ...
    }

### `filter`

The `filter()` method asks all listeners to filter a given value,
passed by the notifier as its second argument, and retrieved by the
listener callable as the second argument:

    [php]
    $dispatcher->filter($event, $response->getContent());

All listeners are passed the value and they must return the filtered
value, whether they altered it or not. All listeners are guaranteed
to be executed.

The notifier can get the filtered value by calling the
`getReturnValue()` method:

    [php]
    $ret = $event->getReturnValue();

Recipes
-------

This section lists some common usage of the event dispatcher.

### Doing something before or after a Method Call

If you want to do something just before, or just after a method is
called, you can notify respectively an event at the beginning or at
the end of the method:

    [php]
    class Foo
    {
      // ...

      public function send($foo, $bar)
      {
        // do something before the method
        $event = new sfEvent($this, 'foo.do_before_send', array('foo' => $foo, 'bar' => $bar));
        $this->dispatcher->notify($event);

        // the real method implementation is here
        // $ret = ...;

        // do something after the method
        $event = new sfEvent($this, 'foo.do_after_send', array('ret' => $ret));
        $this->dispatcher->notify($event);

        return $ret;
      }
    }

### Adding Methods to a Class

To allow multiple classes to add methods to another one, you can
define the magic `__call()` method in the class you want to be
extended like this:

    [php]
    class Foo
    {
      // ...

      public function __call($method, $arguments)
      {
        // create an event named 'foo.method_is_not_found'
        // and pass the method name and the arguments passed to this method
        $event = new sfEvent($this, 'foo.method_is_not_found', array('method' => $method, 'arguments' => $arguments));

        // calls all listeners until one is able implements the $method
        $this->dispatcher->notifyUntil($event);

        // no listener was able to proces the event? The method does not exist
        if (!$event->isProcessed())
        {
          throw new sfException(sprintf('Call to undefined method %s::%s.', get_class($this), $method));
        }

        // return the listener returned value
        return $event->getReturnValue();
      }
    }

Then, create a class that will host the listener:

    [php]
    class Bar
    {
      public function addBarMethodToFoo(sfEvent $event)
      {
        // we only want to respond to the calls to the 'bar' method
        if ('bar' != $event['method'])
        {
          // let the opportnuity to another listener to take care of this unknown method
          return false;
        }

        // the subject object (the foo instance)
        $foo = $event->getSubject();

        // the bar method arguments
        $arguments = $event['parameters'];

        // do something
        // ...

        // set the return value
        $event->setReturnValue($someValue);

        // tell the world that you have processed the event
        return true;
      }
    }

Eventually, add the new `bar` method to the `Foo` class:

    [php]
    $dispatcher->connect('foo.method_is_not_found', array($bar, 'addBarMethodToFoo'));

### Modifying Arguments

If you want to allow third party classes to modify the arguments
passed to a method, just before it is executed, add a `filter` event
at the beginning of the method:

    [php]
    class Foo
    {
      // ...

      public function render($template, $arguments = array())
      {
        // filter the arguments
        $event = new sfEvent($this, 'foo.filter_arguments');
        $this->dispatcher->filter($event, $arguments);

        // get the filtered arguments
        $arguments = $event->getReturnValue();

        // the method starts here
      }
    }

And here is an example of a filter:

    [php]
    class Bar
    {
      public function filterFooArguments(sfEvent $event, $arguments)
      {
        $arguments['processed'] = true;

        return $arguments;
      }
    }
