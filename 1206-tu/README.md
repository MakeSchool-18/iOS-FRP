# Implementing a tiny version of `RxSwift` yourself

Today, we want to understand more about how `RxSwift` works under the hood, by implementing its basic primitives ourselves!

You're not going to need a lot of code for the implementation, but figuring out exactly what to write is going to be somewhat difficult. Below, you'll have relatively precise descriptions in natural language of what each component that you need to implement should do. The information you need might be distributed across the whole page, so your job is to find the right pieces and put them together with your implementation. 

Build and test everything inside of a Playground!


## `RxSwift` concepts

At the bottom of `RxSwift` we have the following three concepts:

- `Event<T>`: an `Event<T>` represents something that is happening inside the application and (potentially) carries a value of type `T` (but can also indicate that something is _completed_ or that an _error_ happened)
- `Observable<T>`: an `Observable<T>` represents a _sequence_ of `Event`s of the same type `T`; you can think of it of being somewhat similar to an `Array`, however it's different in that it _emits_ the latest value that is _appended_ to it by creating an `Event` and wrapping the value inside; once there won't be any more values to be added, it will emit an `Event` to indicate that it _completed_; if anything goes wrong during the lifetime of an `Observable`, it can also emit an _error_ `Event`
- _observer_: an observer is just a function that takes in an `Event<T>` and returns nothing (so, it's return type will be `Void`); an observer can be used to _observe_ what is happening inside of an `Observable`, in particular, the observer is executed every time a new `Event` is being _emitted_ by the `Observable` 


#### Simple Usage

This is how we create an `Observable<Int>` that carries the values `1`, `2`, `3` and then succesfully completes: 

    let numberObservable = Observable<Int>(subscribeHandler: { (observer: (Event<Int>) -> ()) in
      observer(.next(1))
      observer(.next(2))
      observer(.next(3))
      observer(.completed)
    })
  
    numberObservable.subscribe { (event: Event<Int>) in
      print(event)
    }

This code prints:

    next(1)
    next(2)
    next(3)
    completed


#### Implementing `Event`

`Event` should be a _generic_ type, so that it can carry values of any type (e.g. `String`, `Int`, `Bool`,... as well as any custom types). An `Event` should only ever come in one of three forms:

- a `next` event carries a value of the generic type
- an `error` event that carries an `String` object that describes what went wrong
- a `completed` event that doesn't carry an additional value and that simply indicates that the `Observable` that emitted it won't emit any more events

What kind of data structure would work best to fulfill these requirements and what should the implementation look like? Click [this](https://www.google.de/search?q=enums+with+associated+values+swift+3&oq=enums+with+associated+values+swift+3&aqs=chrome..69i57.5687j0j1&sourceid=chrome&ie=UTF-8) only if you can't think of anything yourself.  


#### Implementing a basic version of `Observable`

`Observable` should be a _generic_ type as well. It should have one property called `subscribeHandler` which is a closure which takes an _observer_ function as its argument and returns `Void`. The property is set through an argument that is passed into the class's initializer. The _type_ of the `suscribeHandler` property is `((Event<T>) -> ())) -> ()` (actually, it's `(@escaping (Event<T>) -> ())) -> ()`, but the compiler will tell you itself and for "simplicity" it's left out here). Ouf, that looks scary, but it is exactly what we said before, an _observer_ function takes an `Event<T>` and returns `Void`, and the `subscribeHandler` property is a closure that takes an _observer_ function and returns `Void`. So, `subscribeHandler` is nothing more than a function, that takes another function as an argument.

Yet, at this point it might be worth to create `typealias`es for the function types for observers and the handlers for a subscription (create these inside the `Observable` class so you can use the generic type `T`):

    typealias Observer = (Event<T>) -> ()
    typealias SubscribeHandler = (@escaping Observer) -> ()

So, now it should be easier to write the property `subscribeHandler` of the `Observable` class.

Finally, next to the initializer that takes a `subscribeHandler` as an argument and assigns it to the class' property, `Observable` will have exactly one method called `subscribe`. This method takes one argument of type `Observer`. It only has one line in which it calls the `subscribeHandler` property and passes the `Observer` that it received as an argument.

If you think you're done with the implementation, you can paste the example code from above and see if your implementation works by checking the console and see if it generates the same output as above.


#### Implementing a second initializer for `Observable`

Create another initializer for the `Observable` class that takes an array containing things of type `T` and directly emits `next` events for each value inside the array and finally emits a `completed` event.

You can test your implementation using this code:

    let arrayNumberObservable = Observable<Int>(content: [1,2,3])
    arrayNumberObservable.subscribe { (event: Event<Int>) in
      print(event)
    }

which should print the following to the console:

    next(1)
    next(2)
    next(3)
    completed


#### Implementing `map` and `filter` for `Observable`

At the beginning of this course, we learned how to use the `map` function on Swift `Array`s. Looking at `Observables` as sequences, it makes sense that they also have a `map` function that transforms each element according to some `transform` closure that the map function receives as an argument. 

The `map` function needs to be generic to a new type `U`. It should only take one argument which is a closure that is called `transform`. `transform` itself should take one argument of type `T` and return a value of type `U`. It should be applied to every element inside that is being emitted by the `Observable`.

The difficult part here will be to figure when and how you can access the values inside the events so that you can apply the transform closure on them.

You can test your implementation using this code:

    let arrayNumberObservable = Observable<Int>(content: [1,2,3])
    let stringObservable = arrayNumberObservable.map { (number: Int) in
      return String(number)
    }
    stringObservable.subscribe { (event: Event<String>) in
      print(event)
    }

which should print the following to the console:

    next("1")
    next("2")
    next("3")
    completed


After you're done with `map`, you can create `filter` in a similar manner. The your implementation with the following code:

    let arrayNumberObservable = Observable<Int>(content: [1,2,3])
    let oddNumberObservable = arrayNumberObservable.filter { (number: Int) -> Bool in
      return (number % 2) != 0
    }
    oddNumberObservable.subscribe { (event: Event<Int>) in
      print(event)
    }

which should print the following to the console:

    next(1)
    next(3)
    completed

#### Implementing an extension for `URLSession` to send HTTP request using `Observable`

Much like in the projects that you've been working on in the last couple of weeks (`RxOpenWeatherMapAPI` and `RxGitHubIssuesViewer`), the `Observable` type can also be used to make network requests.

We now want to implement this functionality, too. Build everything inside an `extension` for the class `URLSession` (note that you'll have to `import Foundation` for this).

Implement a function called `response` that takes one argument of type `URL` and returns a new `Observable<URLResponse>`. 

As a hint, note that you have to to return a new `Observable` from this method. You'll need to use the initializer of the `Observable` class (the one you wrote first and that takes the `subscribeHandler` as an arugment). Inside the `subscribeHandler` you can fire off the network request and use the received `URLResponse` to create a `next` event that you are passing to the `Observer` that is the argument for the `subscribeHandler`. If the network request fails, you should emit an `error` event. 

You can test your implementation using this code:

    PlaygroundPage.current.needsIndefiniteExecution = true // enable asynchronous calls in Playground
    URLCache.shared = URLCache(memoryCapacity: 0, diskCapacity: 0, diskPath: nil) // get rid of console warnings
    let responseObservable: Observable<URLResponse> = URLSession.shared.response(url: URL(string:"http://www.makeschool.com")!)
    responseObservable.subscribe { (event: Event<URLResponse>) in
      print(event)
    }

which should print something like the following to the console:

    next(<NSHTTPURLResponse: 0x608000026fa0> { URL: https://www.makeschool.com/ } { status code: 200, headers {
        "Cache-Control" = "max-age=0, private, must-revalidate";
        "Content-Encoding" = gzip;
        "Content-Type" = "text/html; charset=utf-8";
        Date = "Tue, 06 Dec 2016 17:03:31 GMT";
        Server = "cloudflare-nginx";
        Via = "1.1 vegur";
        "cf-ray" = "30d14fc9fdc140be-HAM";
        "x-content-type-options" = nosniff;
        "x-frame-options" = SAMEORIGIN;
        "x-request-id" = "d5aa14b6-9429-407e-b823-84b07277b4d2";
        "x-runtime" = "0.013152";
        "x-xss-protection" = "1; mode=block";
    } })
    completed


#### Implementing an extension for `UIButton` subscribe to touch events using `Observable`

For this part, you'll need some extra code that allows you to add functionality to a button using a closure rather than using the `addTarget:action:for:` method. Copy the following code into a playground before you proceed with the implementation. Note that you'll have to `import UIKit` for this task.

    class ButtonTapHandlerProxy {
      var buttonTapHandler: (UIButton) -> ()
      init(buttonTapHandler: @escaping (UIButton) -> ()) {
        self.buttonTapHandler = buttonTapHandler
      }
    }

    var Key: UInt8 = 0

    extension UIButton {
      
      func block_setButtonTapHandler(buttonTapHandler: @escaping (UIButton) -> ()) {
        objc_setAssociatedObject(self, &Key, ButtonTapHandlerProxy(buttonTapHandler: buttonTapHandler), objc_AssociationPolicy.OBJC_ASSOCIATION_RETAIN_NONATOMIC)
        addTarget(self, action: #selector(UIButton.block_handleButtonTap(sender:)), for: .touchUpInside)
      }
      
      func block_handleButtonTap(sender: UIButton) {
        let wrapper = objc_getAssociatedObject(self, &Key) as! ButtonTapHandlerProxy
        wrapper.buttonTapHandler(sender)
      }

    }

All this code does is to allow you to add functionality to a button touch event using the following syntax:

    button.block_setButtonTapHandler { (sender: UIButton) in
      print("A touch event was received on the button: \(sender)")
    }

Now, we want to implement a function named `tap` that takes no arguments returns an `Observable<Void>` that simply always returns `next` events. It actually never returns `completed` or `error`.

You can test your implementation with the following code:

    import PlaygroundSupport
    let button = UIButton(frame: CGRect(x: 0, y: 0, width: 100, height: 40))
    button.setTitle("Tap me", for: .normal)
    PlaygroundPage.current.liveView = button
    button.tap().subscribe { (event: Event<Void>) in
        print("The button was tapped")
    }

This will enable the `liveView` of the Playground page which allows you to interact with UI elements directly inside the Playground environment. Open the **Assistant Editor** to see the `liveView`. If you click the button that you see in the assistant editor, the following should be printed to the console: `The button was tapped`.












