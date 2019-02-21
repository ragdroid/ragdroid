
# Rxify : The startWith { MVI } pitfall

Sometimes, Rxify spells, when mixed with potions like Kotlin can backfire. I have fallen into this trap, not just once, not even twice but thrice now.

The first time, I spent almost half a day figuring out where I had messed up.

![](https://cdn-images-1.medium.com/max/2000/1*DqjD0HRPkKpceKLwP4S40g.gif)

The second time, I spent almost an hour figuring out what was the issue with my colleague’s reactive chain.

![](https://cdn-images-1.medium.com/max/2000/1*aJx4k3fZp-zwAOO0SlmT0A.gif)

The third and hopefully the last time now, I again spent a few hours trying to figure out why my reactive chain wouldn’t emit anything.

![](https://cdn-images-1.medium.com/max/2000/1*UMjUGCcrD3pmrhf_zOJksQ.gif)

In all the above cases, the culprit was exactly the same. That is when I decided that it is high time now that I should figure out what’s really going on and also share with everyone!

It all *“startedWith”* me taking the next step towards unidirectional reactive architecture. (Don’t worry or maybe do, that there’s not going to be anything about MVI or reactive architecture or Blah in this article, it’s all just { Rx } blown up straight in my face)

Typically with MVI (or reactive arch.) we could have a reactive chain as shown in the snippet below. In my case, I had a chain for my logout flow which looked something like :

    **fun **logout(): Observable<ProfileResult> =** 
       accountService**.logout()
            .map **{ **ProfileResult.LogoutSuccess **as **ProfileResult **}
            **.startWith **{** ProfileResult.LogoutLoading **}**
            .onErrorReturn **{ **ProfileResult.LogoutFailed(**it**.**message**) **}**

We have an accountService which hits the logout api and we are creating a stream of ProfileResult from it. We firstly map the result into a LogoutSuccess result and in order to show a Loading State we need to insert another item in this reactive chain and make it startWith LogoutLoading, so that the very first thing we see on our screen is a loader. And finally, in the case of error we wrap it inside LogoutFailed result.

Can you figure out what’s wrong with the above reactive chain?

Come on! I almost gave it away with the title!

To understand what’s wrong, let’s briefly look at thestartWith() operator

### [startWith](http://reactivex.io/documentation/operators/startwith.html)() operator

This operator concats the supplied source of item(s) before it emits other items of **this** stream.

![[source](http://reactivex.io/documentation/operators/startwith.html)](https://cdn-images-1.medium.com/max/2820/1*VlkiOLOuE2oojMJrfRRkNw.png)*[source](http://reactivex.io/documentation/operators/startwith.html)*

### Overridden methods

**startWith()** operator has the following overridden versions :

* startWith(ObservableSource<**T**> source)

* startWith(T item)

In my case, while going with the flow of map { } and onErrorReturn { }, I ended up using lambda with startWith { } as well. [Read here](https://stackoverflow.com/a/43113495/2854478) for more information on the lambda syntax.

If we look at the method signatures of these three operators, we will notice :

* **startWith**(ObservableSource<**T**> source)

* **map**(Function mapper)

* **onErrorReturn**(Function valueSupplier)

Operators map and onErrorReturn take a function, whereas startWith() actually takes an ObservableSource. ObservableSource is the base interface for any source of items. Classes like Observable extend ObservableSource.

```java
    // from rxjava2

    public interface ObservableSource<T> {
    
        void subscribe(Observer<? super T> observer);
    }
```

In our case, by writing .startWith **{** ProfileResult.Loading **}** we are actually creating an observable source and implementing the subscribe function like :

    **object **TestSource: ObservableSource<ProfileResult>() {
        **override** **fun **subscribe(**observer**: Observer<ProfileResult>) {
            ProfileResult.LogoutLoading
        }
    }

This ObservableSource is good for nothing as we are neither passing through any value to the observer nor emitting any terminal event. *This source never completes!*

Coming back to the definition of startWith(): This operator concats the supplied source of item(s) before it emits other items of **this** stream.

If you are not aware of the concat() operator, you can read about it [here](http://reactivex.io/documentation/operators/concat.html). Concat emits items from the first observable and then waits for it’s terminal event. After first observable completes, it then emits the items from the second observable.

As we saw our source never completes and thus our chain stops emitting any values as it keeps on waiting for the startWith source to emit a value. To correct this, we just need to replace **{ } **with **( ) **or give it a proper implementation of ObservableSource like any type of Observable.

### Corrected Version

    **fun **logout(): Observable<ProfileResult> =** 
       accountService**.logout()
            .map **{ **ProfileResult.LogoutSuccess **as **ProfileResult **}
            **.startWith **(** ProfileResult.LogoutLoading **)**
            .onErrorReturn **{ **ProfileResult.LogoutFailed(**it**.**message**) **}**

### TL;DR

Use lambda’s carefully when using with Rx operators.

Hopefully, I wouldn’t fall into this pitfall again now!

![](https://cdn-images-1.medium.com/max/2000/1*CEY3ugOgZKImY3hUdjKuSw.gif)

![](https://cdn-images-1.medium.com/max/2000/1*WZYmmay7oYOy8NPqSmm-OA.gif)

Thanks to [Julien Veneziano](undefined) and [Ritesh Gupta](undefined) for putting out the fire with me. Thank you for reading :)

Disclaimer: DO try this at home.
