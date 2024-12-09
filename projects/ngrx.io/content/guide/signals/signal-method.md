# SignalMethod

`signalMethod` is a standalone factory function used for managing side effects with Angular signals. It accepts a callback and returns a processor function that can handle either a static value or a signal. The input type can be specified using a generic type argument:

```ts
import { Component } from '@angular/core';
import { signalMethod } from '@ngrx/signals';

@Component({ /* ... */ })
export class NumbersComponent {
  // 👇 This method will have an input argument
  // of type `number | Signal<number>`.
  readonly logDoubledNumber = signalMethod<number>((num) => {
    const double = num * 2;
    console.log(double);
  });
}
```

`logDoubledNumber` can be called with a static value of type `number`, or a Signal of type `number`:

```ts
@Component({ /* ... */ })
export class NumbersComponent {
  readonly logDoubledNumber = signalMethod<number>((num) => {
    const double = num * 2;
    console.log(double);
  });

  constructor() {
    this.logDoubledNumber(1);
    // console output: 2

    const num = signal(2);
    this.logDoubledNumber(num);
    // console output: 4
    
    setTimeout(() => num.set(3), 3_000);
    // console output after 3 seconds: 6
  }
}
```

## Automatic Cleanup

`signalMethod` uses an `effect` internally to track the Signal changes.
By default, the `effect` runs in the injection context of the caller. In the example above, that is `NumbersComponent`. That means, that the `effect` is automatically cleaned up when the component is destroyed.

If the call happens outside an injection context, then the injector of the `signalMethod` is used. This would be the case, if `logDoubledNumber` runs in `ngOnInit`:

```ts
@Component({ /* ... */ })
export class NumbersComponent implements OnInit {
  readonly logDoubledNumber = signalMethod<number>((num) => {
    const double = num * 2;
    console.log(double);
  });

  ngOnInit(): void {
    const value = signal(2);
    // 👇 Uses the injection context of the `NumbersComponent`.
    this.logDoubledNumber(value);
  }
}
```

Even though `logDoubledNumber` is called outside an injection context, automatic cleanup occurs when `NumbersComponent` is destroyed, since `logDoubledNumber` was created within the component's injection context.

However, when creating a `signalMethod` in an ancestor injection context, the cleanup behavior is different:

```ts
@Injectable({ providedIn: 'root' })
export class NumbersService {
  readonly logDoubledNumber = signalMethod<number>((num) => {
    const double = num * 2;
    console.log(double);
  });
}

@Component({ /* ... */ })
export class NumbersComponent implements OnInit {
  readonly numbersService = inject(NumbersService);

  ngOnInit(): void {
    const value = signal(2);
    // 👇 Uses the injection context of the `NumbersService`, which is root.
    this.numbersService.logDoubledNumber(value);
  }
}
```

Here, the `effect` outlives the component, which would produce a memory leak.

## Manual Cleanup

When a `signalMethod` is created in an ancestor injection context, it's necessary to explicitly provide the caller injector to ensure proper cleanup:

```ts
@Component({ /* ... */ })
export class NumbersComponent implements OnInit {
  readonly numbersService = inject(NumbersService);
  readonly injector = inject(Injector);

  ngOnInit(): void {
    const value = signal(1);
    // 👇 Providing the `NumbersComponent` injector
    // to ensure cleanup on component destroy.
    this.numbersService.logDoubledNumber(value, {
      injector: this.injector,
    });
  
    // 👇 No need to provide an injector for static values.
    this.numbersService.logDoubledNumber(2);
  }
}
```

## Initialization Outside of Injection Context

The `signalMethod` must be initialized within an injection context. To initialize it outside an injection context, it's necessary to provide an injector as the second argument:

```ts
@Component({ /* ... */ })
export class NumbersComponent implements OnInit {
  readonly injector = inject(Injector);

  ngOnInit() {
    const logDoubledNumber = signalMethod<number>(
      (num) => console.log(num * 2),
      { injector: this.injector },
    );
  }
}
```

## Advantages over Effect

At first sight, `signalMethod`, might be the same as `effect`:

```ts
@Component({ /* ... */ })
export class NumbersComponent {
  readonly num = signal(2);
  readonly logDoubledNumberEffect = effect(() => {
    console.log(this.num() * 2);
  });
  readonly logDoubledNumber = signalMethod<number>((num) => {
    console.log(num * 2);
  });

  constructor() {
    this.logDoubledNumber(this.num);
  }
}
```

However, `signalMethod` offers three distinctive advantages over `effect`:

- **Flexible Input**: The input argument can be a static value, not just a signal. Additionally, the processor function can be called multiple times with different inputs.
- **No Injection Context Required**: Unlike an `effect`, which requires an injection context or an Injector, `signalMethod`'s "processor function" can be called without an injection context.
- **Explicit Tracking**: Only the Signal of the parameter is tracked, while Signals within the "processor function" stay untracked.

## `signalMethod` compared to `rxMethod`

`signalMethod` is `rxMethod` without RxJS, and is therefore much smaller in terms of bundle size.

Be aware that RxJS is superior to Signals in managing race conditions. Signals have a glitch-free effect, meaning that for multiple synchronous changes, only the last change is propagated. Additionally, they lack powerful operators like `switchMap` or `concatMap`.