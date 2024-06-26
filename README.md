# Decorator Contracts

[![Build](https://github.com/erboladaiorg/officiis-maiores/workflows/Build/badge.svg?branch=master)](https://github.com/erboladaiorg/officiis-maiores/actions?query=workflow%3ABuild%2FRelease)
[![npm version](https://badge.fury.io/js/%40final-hill%2Fdecorator-contracts.svg)](https://www.npmjs.com/package/@erboladaiorg/officiis-maiores)
[![Downloads](https://img.shields.io/npm/dm/@erboladaiorg/officiis-maiores.svg)](https://www.npmjs.com/package/@erboladaiorg/officiis-maiores)

## Table of Contents

- [Introduction](#introduction)
- [Library Installation](#library-installation)
- [Usage](#usage)
- [Checked Mode](#checked-mode)
- [Assertions](#assertions)
- [Implies](#implies)
- [Iff](#iff)
- [Encapsulation](#encapsulation)
- [Invariants](#invariants)
- [Demands](#demands)
- [Ensures](#ensures)
- [Within](#within)
- [Rescue](#rescue)
- [The order of assertions](#the-order-of-assertions)
- [Further Reading](#further-reading)

## Introduction

Decorator Contracts is a library that provides the ability to create and assign
Code Contracts to ECMAScript and TypeScript classes. This enables
enforcement of the
[Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
of [SOLID](https://en.wikipedia.org/wiki/SOLID)
and [Organized Panic](https://en.wikipedia.org/wiki/Exception_handling#Exception_handling_based_on_design_by_contract) to support
[Design By Contract™](https://en.wikipedia.org/wiki/Design_by_contract).

Note that the license for this library is [AGPL-3.0-only](https://www.gnu.org/licenses/agpl-3.0.en.html).
You should [know what that means](https://choosealicense.com/licenses/agpl-3.0/) before
using this. If you would like an exception to this license per section 7
[contact the author](mailto:michael.haufe@final-hill.com).

## Library Installation

The latest version:

`npm install @erboladaiorg/officiis-maiores`

A specific version:

`npm install @erboladaiorg/officiis-maiores@x.x.x`

For use in a browser (no build step) via [unpkg.com](https://unpkg.com/):

```html
<script type="importmap">
{
    "imports": {
        "@erboladaiorg/officiis-maiores": "https://unpkg.com/@erboladaiorg/officiis-maiores"
    }
}
</script>
```

Via [skypack.dev](https://www.skypack.dev/):

`<script type="module" src="https://cdn.skypack.dev/@erboladaiorg/officiis-maiores"></script>`

## Usage

After installation the library can be used as such:

```typescript
import {Contract, Contracted, invariant, checkedMode} from '@erboladaiorg/officiis-maiores';

interface StackType<T> {
    readonly limit: number;
    readonly size: number;
    clear(): void;
    isEmpty(): boolean;
    isFull(): boolean;
    pop(): T;
    push(item: T): void;
    top(): T;
}

const stackContract = new Contract<StackType<any>>({
    [checkedMode]: process.env.NODE_ENV === 'development',
    [invariant](self) {
        return self.isEmpty() == (self.size == 0) &&
            self.isFull() == (self.size == self.limit) &&
            self.size >= 0 && self.size <= self.limit;
    },
    pop: {
        demands(self) { return !self.isEmpty(); },
        ensures(self,old) {
            return !self.isFull() &&
                   self.size == old.size - 1;
        }
    },
    push: {
        demands(self){ return !self.isFull(); },
        ensures(self, old, item) {
            return !self.isEmpty() &&
                self.top() === item &&
                self.size === old.size + 1;
        }
    },
    top: {
        demands(self) {
            return !self.isEmpty();
        }
    }
});

@Contracted(stackContract)
class Stack<T> implements StackType<T> {
    #implementation: T[] = [];
    #size = 0;
    #limit: number;

    constructor(limit: number) {
        this.#limit = limit;
    }

    clear(): void {
        this.#implementation = [];
        this.#size = 0;
    }

    isEmpty(): boolean {
        return this.#implementation.length == 0;
    }

    isFull(): boolean {
        return this.#implementation.length == this.limit;
    }

    get limit() {
        return this.#limit;
    }

    pop(): T {
        this.#size--;

        return this.#implementation.pop()!;
    }

    push(item: T): void {
        this.#size++;
        this.#implementation.push(item);
    }

    get size(): number {
        return this.#size;
    }

    top(): T {
        return this.#implementation.at(-1);
    }
}
```

A contract specifies the semantics of a class and is defined independently from it
the same way that an interface is. Its has the following form:

```ts
const fooContract = new Contract<Foo>({
    [checkedMode]: true,
    [invariant](self: Foo) { return self instanceof Foo },
    methodName: {
        demands(self: Foo, arg1, arg2, argN) { return true },
        ensures(self: Foo, old: Foo, arg1, arg2, argN){ return true }
        rescue(self, error, args, retry) {
            console.error(`Error with given arguments ${JSON.stringify(args)}`)
            console.error(error)
            console.log('Attempting to retry')
            self.value = 7
            retry(...args)
        }
    },
    accessorName: {
        ...
    }
})

const subContract = new Contract<Sub>({
    [extend]: fooContract,
    methodName: {
        demands(self: Foo){ return true }
    }
})
```

## Checked Mode

A contract can be enabled or disabled with the `checkedMode` property.

```ts
const stackContract = new Contract<StackType<any>>({
    [checkedMode]: true,
    ...
})
```

During development and testing you will want to use checked mode. This will
enable all assertion checks. In production all assertion checks become
no-ops for run-time efficiency. As the number of contract definitions can
be numerous, using the appropriate mode becomes increasingly important.

You are not prevented from mixing modes in the event you desire to maintain
a number of checks in a production environment.

This property is optional and defaults to `true`

One approach you could use to manage this is the `env-cmd` package. Details can be [found here](https://www.digitalocean.com/community/tutorials/nodejs-take-command-with-env-cmd). Usage may look like:

```ts
const fooContract = new Contract<Foo>({
    [checkedMode]: !process.env.IS_PRODUCTION,
    ...
})
```

Note: `checkedMode` is set to `false` during evaluation of the assertions and `rescue`. This prevents non-termination when your contract uses other class features during its evaluation. This means you should
avoid mutating state in your assertions, especially the `invariant`, in order to prevent your object
from being put in an invalid state.

## Assertions

Assertions are a fundamental tool for enforcing correctness in an implementation.
They are used inline to express a condition that must evaluate to true at a
particular point of execution.

```typescript
import {assert} from '@erboladaiorg/officiis-maiores';

function avg(xs: number[]): number {
    assert(xs.length > 0, 'The list can not be empty')

    return xs.reduce((sum, next) => sum + next) / xs.length
}
```

If you are using TypeScript `assert` will also assert the type of the condition:

```typescript
let str: any = 'foo';

str.toUpperCase(); // str is any

assert(typeof str == 'string');

str.toUpperCase(); // str is now a string
```

**`assert` should not be used for validating arguments**

Use the `demands` declaration for this purpose.

## Implies

When defining predicates it is a common use case to encode
[material implication](https://en.wikipedia.org/wiki/Material_conditional).

The truth table is as follows:

| *p* | *q* | *p* &rarr; *q* |
|-----|-----|----------------|
|  T   |   T   |  T          |
|  T   |   F   |  F          |
|  F   |   T   |  T          |
|  F   |   F   |  T          |

An example of usage is the encoding of `sunny weather is a precondition of visiting the beach`:

```ts
implies(
    weather.isSunny,
    person.visitsBeach
)
```

This is logically equivalent to: `!p || q`

## Iff

When defining predicates it is a common use case to encode [if and only if](https://en.wikipedia.org/wiki/Logical_biconditional).
Also referred to as a biconditional.

The truth table is as follows:

| *p* | *q* | *p* &harr; *q* |
|-----|-----|----------------|
|  T   |   T   |  T          |
|  T   |   F   |  F          |
|  F   |   T   |  F          |
|  F   |   F   |  T          |

An example of usage is the encoding of `You can ride the train if and only if you have a ticket`:

```ts
iff(
    person.hasTicket,
    person.ridesTrain
)
```

This is logically equivalent to: `implies(p,q) && implies(q,p)`

## Encapsulation

To guarantee invariants remain valid for classes, public property definitions are forbidden.
All interactions with a contracted class must be done through a method or accessor.
This prevents modification of object state outside of the contract system. Private
properties with accessors should be used instead. Example:

```ts
@Contracted(pointContract)
class Point2D {
    accessor x: number
    accessor y: number

    constructor(x: number, y: number) {
        super()
        this.x = x
        this.y = y
    }
}
```

## Invariants

A class is not just a collection of methods it has semantics that
bind them together and the `invariant` declaration describes and enforces these relationships.
This assertion is checked after the associated class is constructed, before and
after every method execution, and before and after every accessor usage (get/set).
If this evaluates to false during class usage an `AssertionError` will be thrown in the library code
as it indicates a bug in the class where it has the [opportunity](#rescue) to handle it.
True assertions do not throw an error. An example of this is given below using a Stack:

```typescript
const stackContract = new Contract<StackType<any>>({
    [invariant](self) {
        return self.isEmpty() == (self.size == 0) &&
               self.isFull() == (self.size == self.limit) &&
               self.size >= 0 && self.size <= self.limit
    }
})

@Contracted(stackContract)
class Stack<T> implements StackType<T> {
    ...
}
```

With the above invariant any attempt to construct an invalid stack will fail:

```typescript
let myStack = new Stack(-1)
```

Additionally, attempting to pop an item from an empty stack would be
nonsensical according to the invariant therefore the following will
throw an `AssertionError` after `pop()` is executed due to the size being negative:

```typescript
let myStack = new Stack(3)
let item = myStack.pop();
```

A contract can be extended and declare an invariant of its own.

```typescript
const baseContract = new Contract<Base>({
    [invariant](self) { return 0 <= self.value && self.value <= 10 }
})

@Contracted(baseContract)
class Base { ... }

const subContract = new Contract<Sub>({
    [extend]: baseContract,
    [invariant](self){ return 10 <= self.value && self.value <= 20 }
})

@Contracted(subContract)
class Sub extends Base { ... }
```

The result is that the invariant of `Sub` is strengthened, meaning that the assertions are and-ed together:

`(0 <= self.value && self.value <= 10) && (10 <= self.value && self.value <= 20)`

Effectively meaning that `self.value` can only be `10`

Only public features have to honor the `invariant`. During execution it can be broken as long as it is restored before exiting.

## Demands

The `demands` declaration describes and enforces an assertion that must be true
before its associated feature can execute. In other words before a client
of your class can execute a method or accessor the defined precondition
must first be met or an error will be raised [to the caller](#the-order-of-assertions).
This is because a failure to meet the precondition indicates a bug in the caller's code
and not yours.

```typescript
const stackContract = new Contract<StackType<any>>({
    ...
    pop: {
        demands: self => !self.isEmpty()
    },
    push: {
        demands: self => !self.isFull()
    }
})

@Contracted(stackContract)
class Stack<T> implements StackType<T> {
    ...
    pop(): T {
        return this.#implementation.pop();
    }
    push(item: T): void {
        this.#implementation.push(item);
    }
}
```

In the above example the precondition of executing `pop`
on a stack is that it is not empty. If this assertion fails
an `AssertionError` is raised.

Static features, including the constructor, cannot be assigned a `demands`
assertion. In the future this may be enabled for non-constructor static methods
but the implications are not clear at present.

If a class feature is overridden then the `demands` assertion still applies:

```typescript
class MyStack<T> extends Stack<T> {
    override pop(): { ... }
}

...
let myStack = new MyStack()

myStack.pop() // throws
```

A contract can be extended and that sub contract can declare a `demands` assertion of its own
for the same feature:

```ts
const baseContract = new Contract<Base>({
    someMethod: {
        demands(){ ... }
    }
})

const subContract = new Contract<Sub>({
    [extend]: baseContract,
    someMethod: {
        demands(){ ... }
    }
})
```

This subcontracted `demands` declaration can not strengthen the `demands` of the base contract.
What this means is that the new precondition will be or-ed with its ancestors. If any `demand` is true
then the obligation is considered fulfilled by the user of the feature.

```ts
const baseContract = new Contract<Base>({
    someMethod: {
        demands(_self, x: number) { return 0 <= x && x <= 10 }
    }
})

@Contracted(baseContract)
class Base {
    someMethod(x: number) { ... }
}

const subContract = new Contract<Sub>({
    [extend]: baseContract,
    someMethod: {
        demands(_self, x: number){ return -10 <= x && x <= 20 }
    }
})

@Contracted(subContract)
class Sub extends Base {
    override someMethod(x: number){ ... }
}
```

In the above example the precondition of `Sub.prototype.someMethod` is:

`(-10 <= x && x <= 20) || (0 <= x && x <= 10)`

## Ensures

The `ensures` declaration describes and enforces an assertion that must be true *after* its associated feature  executes. In other words after a client of your class executes a method or accessor the defined post-condition must be met or an error will be [raised to the library author](#the-order-of-assertions). This indicates a bug in the library code and not in the
caller. The library code then has the [opportunity](#rescue) to capture and fix this error if it's expected.

```typescript
const stackContract<Stack<any>> = new Contract({
    push: {
        ensures(self){ return !self.isEmpty() }
    }
})

@Contracted(stackContract)
class Stack<T> {
    ...
    push(value: T) {
        ...
         this.#implementation.push(value);
    }
}
```

In the above example the post-condition of executing push on a stack is that it is not empty. If this assertion fails an `AssertionError` is raised.

Static features, including the constructor, cannot be assigned an `ensures` declaration. In the future this may be enabled for non-constructor static methods but the implications are not clear at present.

In addition to the `self` argument there is also an `old` argument which provides access to the values of any getters of the instance before its associated member was executed.

```typescript
const stackContract = new Contract<Stack<any>>({
    push: {
        ensures(self, old){ return self.size == old.size + 1 }
    },
    pop: {
        ensures(self, old){ return self.size == old.size - 1 }
    }
})

@Contracted(stackContract)
class Stack<T> {
    ...
    push(value: T) {
         this.#implementation.push(value);
    }
    pop(): T {
        return this.#implementation.pop();
    }
}
```

If a class feature is overridden then the `ensures` assertion still applies:

```typescript
class MyStack<T> extends Stack<T> {
    override push(value: T): { ... }
}

...
let myStack = new MyStack()

myStack.push()
myStack.isEmpty() == false;
```

The remaining arguments of `ensures` reflect the arguments of the associated feature.

```typescript
const baseContract = new Contract<Base>({
    someMethod: {
        ensures(_self, _old, x: number){ return 0 <= x && x <= 10 }
    }
})

@Contracted(baseContract)
class Base {
    someMethod(x: number) { ... }
}
```

A contract can be extended and that sub contract can declare an `ensures` assertion of its own for the same feature:

```ts
const baseContract = new Contract<Base>({
    someMethod: {
        ensures(){ ... }
    }
})

const subContract = new Contract<Sub>({
    [extend]: baseContract,
    someMethod: {
        ensures(){ ... }
    }
})
```

This subcontracted `ensures` declaration cannot weaken the `ensures` of the base contract. What this means is that the new post-condition will be and-ed with its ancestors. If all of the `ensures` are true then the obligation is considered fulfilled by the author of the feature otherwise an `AssertionError` is raised.

```typescript
const baseContract = new Contract<Base>({
    method: {
        ensures(_self, _old, x: number) { return 0 <= x && x <= 10 }
    }
})

@Contracted(baseContract)
class Base {
    method(x: number){ ... }
}

const subContract = new Contract<Sub>({
    [extend]: baseContract,
    method: {
        ensures(_self, _old, x: number){ return -10 <= x && x <= 20 }
    }
})

@Contracted(subContract)
class Sub extends Base {
    method(x: number) { ... }
}
```

In the above example the post-condition of `Sub.prototype.method` is:

`(-10 <= x && x <= 20) && (0 <= x && x <= 10)`

Which effectively means that if the method returns `x === 10`

## Within

The `within` declaration provides a means of requiring the associated feature
to complete execution within a time constraint.

```typescript
const timingContract = new Contract<Spinner>({
    spinLock: {
        within: 100
    }
});

@Contracted(timingContract)
class Spinner {
    spinLock(delay: number) {
        const t1 = Date.now();
        while(Date.now() - t1 < delay) {
            continue;
        }

        return 'Okay';
    }
}

new Ticker().spinLock(50) === 'Okay';

new Ticker().spinLock(500) // throws "Timing constraint violated...";
```

Currently `within` only supports synchronous features. async feature support is planned for a future version

## Rescue

The `rescue` declaration enables a mechanism for providing **Robustness**.
Robustness is the ability of an implementation to respond to situations
not specified; in other words the ability to handle exceptions (pun intended).
This declaration can be associated with class features. The intent of this
is to restore any invariants of the class and optionally retry execution.

```typescript
const stackContract = new Contract<Stack<any>>({
    pop: {
        rescue(_self, error, _args, _retry) {
            console.log(error)
        }
    }
})

@Contracted(stackContract)
class Stack<T> {
    ...
    pop(): T {
        assert(!this.isEmpty(), 'You can not pop from an empty stack')

        return this.#implementation.pop()!
    }
    ...
}
```

In the above naive example if the `pop` method is called when the stack is
empty an exception occurs. The `rescue` declaration intercepts this
exception and handles it by simply logging the error. The exception is
then raised to the caller.

You also have the ability to retry the execution of the decorated
feature from the beginning by calling the `retry` function. This provides
a mechanism for [fault tolerance](https://en.wikipedia.org/wiki/Fault_tolerance).
When `retry` is called the exception will no longer be raised to the caller of the original method.
`retry` can only be called once per exception rescue in order to prevent unbounded
recursion. An example of `retry` usage:

```ts
const studentRepositoryContract = new Contract<StudentRepository>({
    getStudent: {
        rescue(_self, error, [id], retry) {
            console.error(error)
            console.log('Retrying with legacy id...')
            retry(`old-${id}`)
        }
    }
})

@Contracted(studentRepositoryContract)
class StudentRepository {
    ...
    async getStudent(id: string): Student {
        const data = await fetch(`/repos/students/${id}`).then(response => response.json())

        return new Student(data)
    }
}
```

The `rescue` can be defined on the contract of an ancestor class feature and it will be used if none
is defined on the current class:

```ts
const baseContract = new Contract<Base>({
    myMethod: {
        rescue(self, error, args, retry){ ... }
    }
})

@Contracted(baseContract)
class Base {
    ...
    myMethod(){ ... }
}

class Sub extends Base {
    override myMethod(){ ... throw new Error('BOOM!') ... }
}
```

Another capability that the `rescue` declaration provides is
[N-Version programming](https://en.wikipedia.org/wiki/N-version_programming)
to enable [Fault-Tolerance](https://en.wikipedia.org/wiki/Fault_tolerance)
and [Redundancy](https://en.wikipedia.org/wiki/Redundancy_(engineering)).

A dated example of this is performing ajax requests in multi-browser environments where `fetch` may not exist:

```ts
const ajaxRequestContract = new Contract<AjaxRequest>({
    get: {
        rescue(self, _error, [url], retry) {
            self.attempts++
            if(self.attempts < 2)
                retry(url)
            }
    }
})

@Contracted(ajaxRequestContract)
class AjaxRequest {
    #attempts = 0

    get attempts(): number { return this.#attempts }
    set attempts(value: number) { this.#attempts = value }

    get(url) {
        if(this.#attempts == 0)
            return this.getFetch(url)
        else if(this.#attempts == 1)
            return this.getXhr(url)
    }

    async getFetch(url) {
        return await fetch(url)
    }

    async getXhr(url) {
        return await new Promise((resolve, reject) => {
            const xhr = new XMLHttpRequest();
            xhr.open('GET', url);
            xhr.onload = function() {
                if (this.status >= 200 && this.status < 300) {
                    resolve(xhr.response);
                } else {
                    reject({
                        status: this.status,
                        statusText: xhr.statusText
                    });
                }
            };
            xhr.onerror = function() {
                reject({
                    status: this.status,
                    statusText: xhr.statusText
                });
            };
            xhr.send();
        })
    }
}
```

Unlike `try/catch` where exceptions are non-resumable and handled at a location often distantly
and indirectly related to the source, and probably redundantly across the callers,
you can see from the above examples that the `rescue` mechanism enables resumable exceptions and
"[Organized Panic](https://en.wikipedia.org/wiki/Exception_handling#Exception_handling_based_on_design_by_contract)"
where you can lexically determine where the handling occurs and optionally perform changes and retry the feature call.

## The order of assertions

When `obj.feature` is called the happy path is:

![image](./docs/happy-path.png)

If an error is thrown and there is no `rescue` defined then the `invariant`
is checked before the error is raised to the caller.

If an error is thrown in the `invariant` then it is raised to the caller.

If an error is thrown in the `demands` then the error is raised to the caller.
In this case the `invariant` is not checked because the feature body has not
been entered and the assertion cannot modify the state of the class without
calling another method which is governed by its own contracts.

If an error is thrown by the feature body or the `ensures` then
the `rescue` is executed. If `retry` is called then the process
starts from the beginning.

If `rescue` throws an error or does not call `retry` then the
`invariant` is checked before the error is raised to the caller.

![image](./docs/order-of-assertions.png)

## Further Reading

- [Design by Contract](https://en.wikipedia.org/wiki/Design_by_contract)
- [Liskov Substitution Principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle)
- [Object-Oriented Software Construction](https://en.wikipedia.org/wiki/Object-Oriented_Software_Construction)
