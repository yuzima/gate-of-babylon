# Understanding Observable and RxJS

I think many frontend developers have heard about rxjs but they may not familiar with what exactly it is and make use of it in your development. So today I am gonna introduce some basic knowledge about rxjs and it's core concept observable.

## Observable

If you want to learn rxjs, it hard to get around observable which is the basics of rxjs. Rxjs is actually one of the implementations of Observable. There are some other libraries also implement observable, for example, the well-known `core-js` which we all know is used to polyfill our js code, provides `Observable` support.

So what's `Observable`?  The Observable is a type can be used to model push-based data sources such as DOM events, timer intervals, and sockets. Besides, observables are:

- *Compositional*: Observables can be composed with higher-order combinators.
- *Lazy*: Observables do not start emitting data until an **observer** has subscribed.

Basically, the motivation of Observable is it's particularly effective at modeling push-based data stream which originate from environment.

Currently, `Observable` is in the proposal stage for being a standard of ECMAScript. 

### Push & Pull

A push-based data stream is the data stream that producer determines when to send data to the Consumer, the Consumer is unaware of when it will receive that data. While the pull-based data stream is adverse. 

Promises are the most common type of Push system in JavaScript today as Promise is in charge of determining when the value is "pushed" to the callbacks.

### API

After the brief background knowledge of Observable, let's have a look of the API. 

To create a `Observable` needs to pass `SubscriberFunction` to its constructor.  

`SubscriberFunction` is the function defines how `Producer`  supplies data, it's the most important piece to describe the Observable. It receives  a  `SubscriptionObserver` which is a normalized `Observer` which wraps the observer object supplied to subscribe. It has 3 functions: 

- **next():**  Sends the next value in the sequence
- **error():** Sends the sequence error
- **complete():** Sends the completion notification

The following example creates an Observable to emit the string `hi` every second to a subscriber.

```tsx
// This observable will send 1, 2, 3 synchronously then 
// send 4 and complete notification after 1 second.
const observable = new Observable((subscriber) => {
		subscriber.next(1);
	  subscriber.next(2);
	  subscriber.next(3);
	  setTimeout(() => {
	    subscriber.next(4);
	    subscriber.complete();
	  }, 1000);
});
```

Now the observable in the example is created, the code inside `new Observable((subscriber) => { ... })` represents an "Observable execution", but it doesn't execute after creation, to invoke the Observable and see these value, we need to subscribe to it.  This notion is very like function. A function doesn't execute until be called, an Observable doesn't execute until be subscribed. The difference is that observable can deliver values either synchronously or asynchronously.

> Subscribing to an Observable is analogous to calling a Function.
> 

You can chose either to pass an `Observer` or pass callbacks as arguments.

`Observer` is used to receive data from an Observable. It has 4 methods to receive data at different situations.

If you use callbacks, you have to at least pass `onNext` function to receive next value from Observable.

Let's see a simple example of how to subscribe.

```tsx
console.log('just before subscribe');
const subscription = observable.subscribe({
	next(x) { console.log('got value ' + x); },
	error(err) { console.error('something wrong occured: ' + err); },
	complete() { console.log('done'); }
});
console.log('just after subscribe');

/*
just before subscribe
got value 1
got value 2
got value 3
just after subscribe
got value 4
done
*/
```

When `observable.subscribe` is called, the Observer gets attached to the newly created Observable execution. This call also returns an object, the `Subscription`. It represents the ongoing execution and has the minimal API which allows to cancel that execution.

```tsx
// Cancel the ongoing execution.
subscription.unsubscribe();
```

Now we have got the main login of Observable, besides the main logic, there have two static methods `.of` and `.from`

Static method `.of` creates an Observable of the values provided as arguments. The values are delivered synchronously when subscribe is called.

Static method `.from` converts its argument to an Observable, the argument can be either an observable or iterable.

```tsx
// An Observable represents a sequence of values which may be observed.
interface Observable {
    constructor(subscriber : SubscriberFunction);

    // Subscribes to the sequence with an observer
    subscribe(observer : Observer) : Subscription;

    // Subscribes to the sequence with callbacks
    subscribe(onNext : Function,
              onError? : Function,
              onComplete? : Function) : Subscription;

    // Returns itself
    [Symbol.observable]() : Observable;

    // Converts items to an Observable
    static of(...items) : Observable;

    // Converts an observable or iterable to an Observable
    static from(observable) : Observable;
}

// An Observer is used to receive data from an Observable, and is supplied as an argument to **subscribe**.
interface Observer {
    // Receives the subscription object when `subscribe` is called
    start(subscription : Subscription);

    // Receives the next value in the sequence
    next(value);

    // Receives the sequence error
    error(errorValue);

    // Receives a completion notification
    complete();
}

interface Subscription {
    // Cancels the subscription
    unsubscribe() : void;

    // A boolean value indicating whether the subscription is closed
    get closed() : Boolean;
}

function SubscriberFunction(observer: SubscriptionObserver) : (void => void) | Subscription;

interface SubscriptionObserver {
    // Sends the next value in the sequence
    next(value);

    // Sends the sequence error
    error(errorValue);

    // Sends the completion notification
    complete();

    // A boolean value indicating whether the subscription is closed
    get closed() : Boolean;
}
```

## RxJS

As I mentioned before, rxjs is a library implement Observable, `Observable` is its core type, but it provides more than ECAMScript proposal.

The essential concepts in RxJS are:

- **Observable:** represents the idea of an invokable collection of future values or events.
- **Observer:** is a collection of callbacks that knows how to listen to values delivered by the Observable.
- **Subscription:** represents the execution of an Observable, is primarily useful for cancelling the execution.
- **Operators:**
    
    Are pure functions that enable a functional programming style of dealing with collections. There are two kinds of operators:
    
    - **Creation Operators:**  can create new Observable, the static methods we learnt in Observable proposal `of` and `from` are two creation operators. For example  `of(1, 2, 3)` will create an Observable emitting 1, 2, 3 and `from([10, 20, 30])` will convert an array to Observable emitting  10, 20 and 30 in sequence**;**
    - **Pipeable Operators:**  are pure functions which can be used to pipe Observables, include `map`,  `filter`, `mergeMap`. They don't change the existing Observable, instead, they return a new Observable.
    
    You may noticed that the operators in rxjs looks very familiar to lodash. You can consider rxjs is the lodash for push-based event stream.
    
- **Subject:** is the equivalent to an EventEmitter, and the only way of multicasting a value or event to multiple Observers.
- **Schedulers:** are centralized dispatchers to control concurrency, allowing us to coordinate when computation happens on e.g. `setTimeout` or `requestAnimationFrame` or others.

After understanding the core concepts, you may question how exactly does rxjs benefit our programs? I'd like to demonstrate some instances which may be more straight.

First example shows how to make use of operator to isolate the state. 

```tsx
let count = 0;
document.addEventListener('click', () => console.log(`Clicked ${++count} times`));
```

```tsx
import { fromEvent, interval } from 'rxjs';
import { scan } from 'rxjs/operators';

const startButton = document.querySelector('#start');

const fromEvent(startButton, 'click')
	// scan applies an accumulator function over the source Observable
	.pipe(scan(count => count + 1, 0))
	.subscribe((count) => count => console.log(`Clicked ${count} times`));
```

As the pipeable operators are pure functions, every pipeable operator takes one observable and returns another observable, so they are composable. You can pipe many pipeable operators to create a readable operators pipeline by putting them inside `pipe()`.

```tsx
obs.pipe(
  op1(),
  op2(),
  op3(),
  op3(),
)
// As a stylistic matter, op()(obs) is never used, even if there is only one operator; 
// obs.pipe(op()) is universally preferred.
```

Now we have seen that we can put multiple operators into one Observable's pipeline, can we combine multiple Observables into one data stream. Yes we can. RxJS provides an API called merge which can achieve that. Let have look of the its marble diagram.

It's pretty straight how `merge` works, it maps values from both Observables into final data stream.

```tsx

```

- combineLatest: 买入价卖出价

Using rxjs, another useful thing you must know is `Subject` . When using Observable, each subscribed Observer owns an independent execution of the Observable.  So you can't multicast one Observable execution to many Observers. But you can do it by `Subject`. 

This example shows two observers share one "multicasted Observable". 

Multicasted Observable passed notifications through a Subject which may have many subscribers. 

The Observers subscribe to an underlying Subject, and the Subject subscribes to the source Observable. 

```tsx
import { interval, Subject } from 'rxjs';
import { multicast, refCount } from 'rxjs/operators';

const source = interval(500);
const subject = new Subject();

// refCount's function is to invoke the observable immediately when the observable is subscribed 
// by any observer and stop execution when all observers have unsubscribed.
const refCounted = source.pipe(multicast(subject), refCount());
let subscriptionA, subscriptionB;

console.log('observerA subscribed');
subscriptionA = refCounted.subscribe((v) => console.log(`observerA: ${v}`));

setTimeout(() => {
	console.log('observerB subscribed');
  subscriptionB = refCounted.subscribe((v) => console.log(`observerB: ${v}`));
}, 600);

setTimeout(() => {
  console.log('observerA unsubscribed');
  subscription1.unsubscribe();
}, 1200);
 
// This is when the shared Observable execution will stop, because
// `refCounted` would have no more subscribers after this
setTimeout(() => {
  console.log('observerB unsubscribed');
  subscription2.unsubscribe();
}, 2000);

// Logs
// observerA subscribed
// observerA: 0
// observerB subscribed
// observerA: 1
// observerB: 1
// observerA unsubscribed
// observerB: 2
// observerB unsubscribed
```

Regarding Subject, there is very useful variation `ReplaySubject`. The function of `ReplaySubject` is when a new observer subscribe to the Subject, it can get the old value immediately instead of waiting for the new value to be emitted. For convenience, rxjs provides an operator `publishReplay` which represents `multicast(new ReplaySubject())`.

> publishReplay() === multicast(new ReplaySubject())
> 

`publishReplay()` returns `ConnectableObservable` so we can still use `refCount()` to manage connections. This feature can be used for caching HTTP response.

```tsx
import { Observable } from 'rxjs';
import { fromFetch } from 'rxjs/fetch';
import { map, publishReplay, refCount } from 'rxjs/operators';

export interface Config {
    componentType: string, 
    show: Boolean
}

export class ConfigService {
    configs: Observable<Config[]>;
    constructor() { }

    // Get configs from server | HTTP GET
    getConfigs(): Observable<Config[]> {

        // Cache it once if configs value is false
        if (!this.configs) {
            this.configs = fromFetch('/configs').pipe(
                map(data => data['configs']),
                publishReplay(1), // this tells Rx to cache the latest emitted
                refCount() // and this tells Rx to keep the Observable alive as long as there are any Subscribers
            );
        }
        return this.configs;
    }

    // Clear configs
    clearCache() {
        this.configs = null;
    }
}
```

Next example I want to show you how to use `Observable` handles vast requests. If you have array has over 10,000 IDs and you need to send request for each of them as the parameter. Of course we can't send them at the same time because it will block the network and the browser also doesn't allow you to do that. By `mergeMap` operator in rxjs we can map each source value to an Observable which is merged in the output Observable and use concurrent parameter to control how many requests are sent at the same time.

```tsx
import { of } from 'rxjs';
import { mergeMap } from 'rxjs/operators';

const concurrent = 10;
const userIds = [...new Array(100)].map((_, i) => i + 1);
const userIds$ = of(...userIds);

const fetchUser = (userId) => {
    console.log(`fetching ${userId}`);
    return new Promise((resolve) => setTimeout(() => {
        resolve({ userId })
    }, (Math.random() + 1) * 300));
}

userIds$.pipe(
    mergeMap(value => fetchUser(value), concurrent),
).subscribe(user => {
    console.log(user);
});
```

When sending hundreds and  thousands requests, it's very common that some of them may be blocked and keep in the pending status leading to the result that these Observables can't be completed and keep occupying the concurrent number. The solution is adding a `timeout` operator to the pipeline to restrict the maximum waiting time. If one request overs the timeout time, it will throw a TimeoutError which can be caught and retry sending again. 

After these examples, probably you are still doubting the value of using rxjs because at least we still can achieve these functions by other ways.

1. Purity
2. Flow
3. Values