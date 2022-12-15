---
title: "Angular anti-patterns #01: Subscribing to every observable"
date: 2022-11-17
tags:
  - angular
  - angular-anti-patterns
---

Rxjs is probably the most difficult thing to grasp when starting out with angular.
For that reason often times the benefits of rxjs are not used and code like the following is written.

```typescript
@Component({
  selector: 'app-subscribing-to-everything-bad',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h3>Bad</h3>
    <p>{{result}}</p>
  `
})
export class SubscribingToEverythingBadComponent {
  result: string | undefined;

  constructor(someService: SomeServiceService) {
    someService.loadSomething().subscribe((result) => {
      this.result = result;
    });
  }
}
```

The first thing that comes to ones mind when seeing observables would be to subscribe to get the value out of there.
Because then you can treat it like any normal variable without needing to learn anything about rxjs.
And as in this example this actually works as expected. But lets continue on and we will see why we want to avoid that kind of code.


## Problem

Lets image the service call now needs an id from the url to fetch the needed data.
No problem, lets inject the activatedRoute and get the parameters from there.

```typescript
@Component({/** same as before **/}
export class SubscribingToEverythingBadComponent {
  result: string | undefined;
  private id: string | undefined | null;

  constructor(someService: SomeServiceService, activatedRoute: ActivatedRoute) {
    activatedRoute.queryParamMap.subscribe(map => {
      this.id = map.get('id');
    });
    someService.loadSomething(this.id!!).subscribe((result) => {
      this.result = result;
    });
  }
}
```

Ok, so now we read the id parameter from the activated route `queryParamMap` attribute which happens to be another observable.
Then we are using that id as a parameter to our service call.
It does not take a long time to find out that this **does not work**. And this is where people unfamiliar with rxjs and angular often times get confused.
I have been there too.

One could come to the conclusion that the subscribe is not executing *fast* enough before the service is called and come up with the following code.

```typescript
@Component({/** same as before **/}
export class SubscribingToEverythingBadComponent {
  result: string | undefined;
  private id: string | undefined | null;

  constructor(someService: SomeServiceService, activatedRoute: ActivatedRoute) {
    activatedRoute.queryParamMap.subscribe(map => {
      this.id = map.get('id');
    });
    setTimeout(() => {
      someService.loadSomething(this.id!!).subscribe((result) => {
          this.result = result;
      });
    }):
  }
}
```

With that change the code will work again as expected, at least in the simple case of the first page load.
But if we happen to change the route parameter without reloading the page the service will not be called with the changed id and we will get inconsistent results. And all of that comes from the fact that we try to **work against the asynchronous nature of observables** by subscribing anytime we have the chance to do so.

## Solution
We can get rid of these problems if we do not subscribe to everything and instead work with the powerful utilities that rxjs offers.

```typescript
@Component({
  selector: 'app-subscribing-to-everything-good',
  standalone: true,
  imports: [CommonModule],
  template: `
    <h3>Good</h3>
    <p>{{result | async}}</p>
  `
})
export class SubscribingToEverythingGoodComponent {

  result: Observable<string>

  constructor(someService: SomeServiceService, activatedRoute: ActivatedRoute) {
    this.result = someService.loadSomething();
  }
}
```

Instead of subscribing to the Observable ourselves we use the async pipe to let angular take care of subscribing. This has additional advantages when it comes to long lived observables that automatically gets unsubscribed when the component is destroyed.
So far the result of the code is the same as in the first code example.

If we continue to add the id from the url parameters we will see the benefits we get.

```typescript
@Component({/** same as before **/}
export class SubscribingToEverythingGoodComponent {

  result: Observable<string>

  constructor(private someService: SomeServiceService, private activatedRoute: ActivatedRoute) {
    this.result = activatedRoute.queryParamMap.pipe(
      switchMap(map => this.someService.loadSomething(map.get('id')!!))
    )
  }
}

```

Instead of subscribing to the activated route we use the `switchMap` operator to use the id for our service call.
The result will be an obersable that emits every time the activated route changes and the data is loaded from the service.

This will now also work when the route changes and give us consistent results.

> Of course it sometimes is totally valid to subscribe to observables in your component code, but I would advise to do so rarely and always think about if it is really needed as it often time leads to code that works against rxjs.


