# Advanced features of Flowtype

By: Victor Powell

---

### Specify the Flow version explicitly

````
[version]
0.104.0
````

Note: Configure your flowconfig file to use a specific version of Flow. This will help avoid common version mismatch issues, especially for IDEs that may attempt to use their own built in version of Flow instead of the Flow binary in your project.

---

### Import only what you need

````
[ignore]
<PROJECT_ROOT>/node_modules/.*
````

Note: Flow, by default, will attempt to parse every file in your `node_modules` directory. For even medium sized projects, this could be several hundred thousand(!) files. Most of these are unlikely to be published with types or their types live in the [flow-typed](https://github.com/flow-typed/flow-typed) repo.

---

### Explicit imports

````[ignore]
# Ignore all modules...
<PROJECT_ROOT>/node_modules/.*
# except these ones.
!<PROJECT_ROOT>/node_modules/baseui/.*
````

Note: To explicitly add a dependency, use `!` in the `[ignore]` directive to specify an exception.

---

### `dist` directories

````
[ignore]
# ignore `dist` directory.
.*/dist/.*
````

Note: Projects often also have `dist` style directories that Flow should never parse. You can also exclude those in a similar way.

---
### Use `flow ls`

````sh
$ flow ls
foo/bar/index.js
node_modules/wow/index.js
````

Note: To debug what files Flow is parsing, you can use the `flow ls` command. (through yarn: `yarn flow ls`)

---

### Use `lints`

````
[lints]
untyped-type-import=error
untyped-import=error
````

Note: If we exclude all modules types by default, we'll also want to have a way to detect, if we're adding a new dependency, that we're not picking up any types for it. To have Flow error for imports that don't have types, use the following lint rules:

---

### type stubs

````sh
flow-typed create-stub mydep@1.x.x
````

Note: If you're just adding these rules in your project, you'll see lots of errors for untyped imports. For these, we recommend you either install the libdef from flow-typed or add a type "stub".

---

### clean things up

````diff
declare module 'flat' {
  declare module.exports: any;
}
- declare module 'flat/cli' {
-  declare module.exports: any;
- }

- declare module 'flat/test/test' {
-   declare module.exports: any;
- }
````

Note: Since these imports were previous untyped anyway, there's no additional lose of type converage. We're just making the untyped code explicitly untyped instead of implicitly untyped. That said, be careful. flow-typed might create very large stub files that have an entry for each file in the module. In most cases, you can shorten these to just a single `declare`.

---

### Avoid `create-stub` for new libdefs

Note: As you add new libdefs that dont already exist in flow-typed, it's better to avoid flow-typed create-stub and stub out your definitions as you need them manually. Then add the new definitions for functions or features are needed. Once your definitions reach some level of maturity, you can share them back to the flow-typed repo.

---

###  Use [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped) as a guide for new libdefs

Note: If you can't find a type definition on flow-typed, you can also usually copy / paste the definition from the TypeScript repo and make relatively small changes to get it working.

---

### Avoid "dynamic" libraries like `dotty()`

Note: Try to avoid libraries with dynamic utilities. For example: `get(obj, '.foo.bar')`. Because the semantics are dynamic and can only be determined at runtime, it's difficult (impossible?) for any type checker to understand this at compile time. One example of a library with lots of dynamic features is lodash.

---

### Use other lint rules

````js
const x: ?number = 5;
if (x) {} // sketchy because x could be either null or 0.
````

Note: Use `sketchy-null` to detect common issues where someone may accidently compare a string or number to a false.

---

### `strict` and `strict-local`

Note: For existing projects, it's usually not practical to turn on all possible checks. For this reason, you can use the `[strict]` directive to add additional type checks for files that specify `// @flow strict` or `// @flow strict-local`. The difference between the two is that strict-local will only apply to your file where as `@flow strict` also requires that any files that file depends on is also strict.

---

````
[strict]
nonstrict-import
unclear-type
unsafe-getters-setters
inexact-spread
deprecated-call-syntax
untyped-import
untyped-type-import
sketchy-null
unnecessary-invariant
sketchy-number-and
deprecated-type
````

---

### `unclear-type`

If you're going to add any lint check, the most important of these is probably `unclear-type`. It will eror if you use an `any`.

---

#### The difference between spread and `&`

`...` is usually want you want.

````js
type A = { prop: string };
type B = { prop: number };

({ prop: 'foo' }: A & B); // Error!
({ prop: 10 }: A & B); // Error!
({ prop: 'foo' }: {...A, ...B});
({ prop: 10 }: {...A, ...B});
````

[[Try Flow]](https://flow.org/try/#0PTAEAEDMBsHsHcBQiAuBPADgU1AQVALygDeoGATrBgFygDOK5AlgHYDmoAvgNyqY4AhQiTKUaoFgFcAtgCMs5Lr0QAKUhSq0A5JFiwtXWvgBkoAQEpuoEKACi5SuQCEq9WNoBGAAyG8oUxZWNvaOLmqimqA6egactMQAdEm4ADSgSQkCnJauEeLevonJaRlZlkA)

---

#### You checked if `thing` was not `undefined` but Flow still complains

````js
type Person = {| thing: string | void |};

const me: Person = { thing: 'Ive got value.' };

if (me.thing) {
  doAThing();
  print(me.thing); // Error!
}

function doAThing() {}
function print(message: string) {}
````

    Cannot call `print` with `me.thing` bound to `message`
    because undefined [1] is incompatible with string [2].

[[Try Flow]](https://flow.org/try/#0C4TwDgpgBAChBOBnA9gOygXigbwD5WAAsBLVAcwC4pFh5Syp8A3ZYgE0YF8BuAKF4DGaGlAC2EKnCRpMOAiXJUA5AEkm0MsmBQmAQwA2AVwgA6JVB79iAMygAKcSaL0AlDl5QobZAEEAKgpkdi58nmB0qMAOps7kIbyc-NaGqALAxDLe-oHBOInJqeky4aRR4oiIumQS1LSueUA)

Note: This is because Flow doesn't know for sure `doAThing()` doesn't unset `me.thing` to be `undefined` again. Another way to say this is that the refinement of `me.thing` is lost because of the call to some unrelated function that might impact the refinement. One way to fix this is to assign the value to a constant.

---

````diff
  if (me.thing) {
+   const { thing } = me;
    doAThing();
-   print(me.thing);
+   print(thing);
  }
````

---

````diff
  if (me.thing) {
-   doAThing();
    print(me.thing);
  }
````

Note: or remove the function triggering the refinement loss.

---

#### Unsealed objects are not safe

````js
const foo = {};
foo.bar(); // No error!
````
[[Try Flow]](https://flow.org/try/#0MYewdgzgLgBAZiEMC8MDeBfA3AKASAOgCMBDAJwAoBKLGAejpgDkkBTMskMgQiA)

Note: A somewhat misunderstood feature of Flow is that of unsealed objects.

---

````js
const foo = {};
foo.bar = 10;
foo.bar(); // Error!
````

Note: This isn’t quite the same as `any`. For example the following errors correctly.

---

````js
const foo: { bar?: () => null } = {};
foo.bar(); // Error!
````

Note: Flow is attempting to track types of newly added properties. Although this is a nice feature when first migrating an untyped project to Flow, it can cause issues later. Instead of using unsealed objects, when practical, you should explicitly type empty object declarations. It seems likely that a future version of Flow may remove unsealed objects. In the meantime, to avoid issues with unsealed objects, if you want to create an empty object that is sealed, you can use `{...undefined}`.

---

#### Use `$ReadOnly` to avoid mutations

````js
// @flow

// Definitely has `foo`.
const b: {| foo: string |} = { foo: 'foo' };

function func1(a: {| foo: string | void |}) {
  // This signature says "I might set `a.foo`
  // to undefined."
}

func1(b); // Error!

````

---

````js

// Definitely has `foo`.
const b: {| foo: string |} = { foo: 'foo' };

function func2(a: $ReadOnly<{| foo: string | void |}>) {
 // This signature says "I wont modify touch the
 // properties of `a`."
}

func2(b); // Works!
````

[[Try Flow]](https://flow.org/try/#0PTAEAEDMBsHsHcBQiDGsB2BnALqARgFygDeAPqJLLETgE4CW6A5qKQL6gC8JFVRA5JVj9QbANygQoACIBTSI3rZZ0AJ6gAFgENMoAAZC9AOmSQAruhTZ6GChZQBGABRaiZXtVB1GLcgDdYegATVjYAShJEUEkwABUNel1MeiZ0LWwzWlkvLVVdACIASVAAWxSNXExZXD0tI0NQbFhQCyD5Rlkgo3zENlN7ZzwwiSkAUVpaWFoAQn7La1tzSwAmFyIAEgAlWS0ggHl0NQAedyEabAZmVlAA4NCAPgjiKKl4xK8UtIysnLzQItA8AwuBKsCC9Eg6iaZhQGkaGmyAAdJojZLRrLJdLBIPotMYen1EEsUKshiMwAB1KYAa0wsyAA)

---

#### Flow does not enforce existance on array index access

````js
const a: Array<{|foo: string|}> = [];
const b: string = a[0].foo; // `b` is actually undefined!
````

[[Try Flow]](https://flow.org/try/#0MYewdgzgLgBAhgLhgQQE6rgTwDwG8A+AZiCEtKgJZgDm+AvgHwwC8MA2gLoDcAUKJLABGZKJRot4bAAwcAdMRBcYAemUwABoPUwKEeMCgBXOABsTmGIbAATAKaEqt6wEIgA)

---

#### 'Coverage' can be helpful but don't let it mislead you

<img width="494" alt="Screen Shot 2019-08-13 at 12 17 51 AM" src="https://user-images.githubusercontent.com/583385/62922256-d4b9cd00-bd5f-11e9-8c07-063e6b05c454.png">


---

That said, there was recently added experimental support for "trusted" coverage.

<img width="959" alt="Screen Shot 2019-08-12 at 10 26 45 PM" src="https://user-images.githubusercontent.com/583385/62917516-6d951c00-bd51-11e9-9b7e-de990457c17f.png">
