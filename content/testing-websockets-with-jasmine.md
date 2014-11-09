Title: Test JavaScript WebSockets with Jasmine using Depency Injection
Date: 2014-11-09
Category: testing
Tags: testing, jasmine, depency injection
Slug: testing-websockets-using-dependency-injection
Authors: Auke Willem Oosterhoff
Summary: Testing your code that usesn JavaScript WebSockets can done easily with Jasmine's Spies and Dependency Injection.

For my project [Goppetto][goppetto] I needed some event handler that could processes messages received over a websocket from some
externel server. The JSON formatted messages looked liked this:

    #!json
    {
        "event": "set_pin",
        "data": {
            "pin_id": 1,
            "state": "low"
        }
    }

My very first approach to handle incomming events looked somewhat like below.
I created a `WebSocket` and attached a listener to `WebSocket.onmessage`. This simple
handler fetches the event name out of the message and calls a function based on the name of the event.

    #!javascript
    ws = new WebSocket('ws://some_url.com/socket');

    ws.onmessage = function(message) {
        var eventName = message.data['event'];
        var eventData = message.data['data'];

        if (eventName === 'set_pin') {
            setPin(eventData);
        } else if(eventName === 'other_event') {
            otherEvent(eventData);
        }
    };

This code worked and I was happy. But then I asked myself: "How can I test these lines?".
A `WebSocket` requires a valid url, creating a `WebSocket` without existing url fails:

    :::javascript
    WebSocket connection to 'ws://some_url.com/socket' failed: 
      Error during WebSocket handshake: Unexpected response code: 200

During my test run I can't rely on some external server to connect the `WebSocket` with. And even if it was possible to
start such a service only for my tests I didn't want that. Assume I was able to use a WebSocket in my tests.
To test the dispatches I had to send a message via the `WebSockets`s `send` method to to receiving end of the socket and check if
the correct callback is called with the expected parameters. But than I would test two things. I would test whether the socket's `send` function
works correctly and I would test if my dispatcher works correctly. But I only want tot test one thing, I want to test if my dispatcher works.

So I had to rewrite my little dispatcher in such a way I that it didn't need a `WebSocket`.

The solution is quite easy and common and it's called: [Dependency Injection][dependency-injection]. I can't explain the concept better than Wikipedia does,
so read that article when you are not familiar with the term. But I want to quote a fragment:

> "The result is clients that are more independent and that are easier to unit test in isolation using stubs or mock objects that simulate other objects not under test. This ease of testing is often the first benefit noticed when using dependency injection."

Based on [Ismael Celis][event-dispatcher]'s sample (I got to it while reading [this][article] post on his site) I rewrote the code above and created an `EventDispatcher` which requires a `WebSocket` during initialisation.

    #!javascript
    var EventDispatcher = function(socket) {
        var dit = this;
        this.socket = socket;
        this.callbacks = {};

        this.bind = function(event_name, callback) {
            this.callbacks[event_name] = this.callbacks[event_name] || [];
            this.callbacks[event_name].push(callback);
            return this;
        };

        var socket.onmessage = function(event) {
            var json = JSON.parse(event.data);
            dispatch(json.event, json.data);
        };

        var dispatch = function(event_name, event_data) {
            var callbacks = dit.callbacks[event_name];
            if (typeof callbacks === 'undefined') {
                return;
            }

            for (var i = 0; i < callbacks.length; i++) {
                callbacks[i](event_data);
            }
        };
    };

The `EventDispatcher` can be used as follows:

    #!javascript
    var ed = EventDispatcher(new WebSocket('ws://some_url.com/socket'));
    ed.bind('pin_state', pinState);
    ed.bind('other_callback', otherCallBack);

Testing is easy now, as you see below.
Instead of using `WebSocket` for testing I use [Jasmine Spies][jasmine-spies] and inject this `Spy` into
th `EventDispatcher`. On line 11 I created a callback. I used a `Spy` so I can use the matcher `toHaveBeenCalledWith`.
On line 12 mock a message for the event `set_pin`. After binding the callback to the event I called `socket.onmessage()`
and check if the callback has been called as expected.

    #!javascript
    describe('An EventDispatcher', function() {
        var socket;
        var ed;

        beforeEach(function() {
            socket = jasmine.createSpy('socket');
            ed = new EventDispatcher(socket);
        });

        it('should dispatch events when it receives them.', function() {
            var callback = jasmine.createSpy('callback');
            // Mock of MessageEvent aka the message send over the socket.
            var event = {'data':
                JSON.stringify({
                    'event': 'pin_state',
                    'data': {
                        'pin_id': 3,
                        'state': 0
                    }
                })
            };

            ed.bind('pin_state', callback);
            socket.onmessage(event);
            expect(callback).toHaveBeenCalledWith({'pin_id': 3, 'state': 0});
        });
    });


[goppetto]: https://github.com/OrangeTux/Goppetto "Source of Goppetto"
[dependency-injection]: http://en.wikipedia.org/wiki/Dependency_injection "Wikipedia page about Dependency Injection"
[event-dispatcher]:https://gist.github.com/ismasan/299789
[jasmine-spies]:http://jasmine.github.io/2.0/introduction.html#section-Spies "Jasmine documentation about Spies"
[article]: https://www.new-bamboo.co.uk/blog/2010/02/10/json-event-based-convention-websockets/ "A JSON event-based convention for WebSockets"
[message-event]: https://developer.mozilla.org/en-US/docs/Web/API/MessageEvent "Mozilla Developer Network about MessageEvent"
