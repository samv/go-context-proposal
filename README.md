# Title:

proposal: Go 2: context variables

# Other Languages:

Go, Python, JS, C, C++, Perl, Java, Groovy, Pascal, Assembly

# Has this idea, or one like it, been proposed before?

Not officially, AFAICT.  I have submitted this to the Golang-nuts list once in 2017, and once more recently.

# Proposal

## Context Variables

A variable could be marked as "context" such as with `context var Context Partial` (example syntax for illustration purposes).  Unlike a regular variable declaration via `var`, the variable is initialized if it has been initialized anywhere in the call stack.  If no such contexts exist, then the variable is considered to have the zero value for the declared type.

Assignments to a context variable will have the effect of setting or changing the value in the current scope and any child scopes.  The change will be visible in any child scope which inherits the new context.

### Context Variable visibility

Context variables would have slightly different visibility rules, which should hopefully follow from what they are.

#### Cross–package context

The appearance _anywhere_ in a package, top–level or in a function, defines the _exported_ context variable `Context`:

    context var Context Partial

Say the package is called `logging`.  Other packages that import `logging` could then refer to this variable as `logging.Context`.  They could also set the value, presumably in this case with a partial log line, containing relevant IDs that are to be added to all scopes below (example: transaction IDs, user IDs).

Assigning to the variable takes on the new syntax (illustrative):

    context var logging.Context = logging.Context.With("userID", userID)

In this case, the value on the right sees the old value of the `logging.Context` variable, the value prior to the assignment.  In this usage, `logging.Partial` must be a pointer type, with a `With` method which allows a `nil` receiver and returns a modified `logging.Partial`, for this to safely "bootstrap" without requiring setup code.

#### Private context

Private context variables use a lowercase first letter of their name.

    context var currentUser iam.User

In this case, the variable name `currentUser` can only be accessed and set within the package the declaration is in, but it retains the dynamic scope regardless of where in the package it is defined.

### Context funcs

In addition to the 'var' keyword, the 'func' keyword can also be prepended with 'context'.  This declares a function or a method, called as normal, but with the modification that any changes the function makes to context variables are updated in the caller's context on return to that function.

This can be used to make helper functions that set up several context variables at once, without having to write lots of boilerplate/repeated code in your functions.  It can also be used for writing backwards compatible APIs, where you are migrating from a `package.WithContext(context.Context)`–style API to `context var` variables.

### Defining "child context"

In some cases, it is necessary to clarify exactly where the context "inherits" from.  For simple function calls to named functions and methods, as well as new goroutines, there is only one caller context from which scope can search, so that is used.

However, there are two cases where the "parent context" is potentially ambiguous: closures and bound method calls.  In order to provide the most flexibility when working with context variables, and cause the least possible surprise, the rule is: bound method calls _do not_ capture the context in which the method is bound with the instance, while closures _do_.

This means that the only context for a bound method call is the instance that is bound, and all dynamic scoped/context variables inside that method call are taken from the context that the bound method func is eventually called from.

By contrast, a closure by convention closes over everything by default: all variables in scope, which includes the dynamically scoped variables.

Where useful, to break ties context should be thought of as a read–only, append–only mutable state map, keyed by package and symbol, referenced by a pointer which is stored in a virtual register with a callee–save calling convention.  As may be clear, `context func`'s simply do not restore the context register.

### Standard Library Changes

In general it is proposed to change the context methods `Deadline()`, `Err()` and `Done()` to `context var` variables/callables in the `sync.` or `time.` namespace(s).  The hope is that this will in general be simpler and fewer lines of boilerplate to make work.

For backwards compatibility's sake, standard library functions that relate to context can be marked `context` to indicate they do not restore the caller's context on return.  The caller sees no difference to the function signature.  This can be used by new API calls to maintain a `context.Context`, and by old API calls to set the new API's variableZs.

While not a part of the language, seeing the issues that the stdlib functions have with context changes will help explore the problem space of moving from `context.Context` to less monolithic state.

### `context` library

The `context.Context` methods `Deadline()`, `Done()` and `Err()` are replaced by `context var`'s `time.Deadline`, `sync.Done` and `sync.Err()`, respectively.  Depending on the exact syntax used, `context.Context` can pass through to the `context var` values instead of explicitly saving them.

`context.WithCancel`, for instance, would be a `context func` method, and redefines the `sync.Done` (and `sync.Err`) context var inside, wraps the passed–in context and returns a cancellation callback.  This would allow a smooth upgrade path, and code which is ready can use the new API (described below) while the old API works with the new one.

Of course, the `context` package functions like `WithCancel`, `WithDeadline` and `WithTimeout` still return a `context.Context` instance, which does nothing especially magic or different from before, for backwards compatibility.

These compatibility functions are declared as `context func ...` and both set the new variable name and an implicit `context var` for the context; this cannot be read directly, but when you use `context.TODO()`, it is returned.  This means in your new code, you can use `context var` and then switch back to libraries that require `context.Context` using `context.TODO()` as the first argument.

### `sync` library

#### `context func sync.Canceler()`

For dynamic scope/context–based cancellation, call `sync.Canceler()`, which is called by `context.WithCancel`

    canceler := sync.Canceler()

This function as a side effect sets `sync.Done` and `sync.Err` in the calling scope's dynamic context.  Any function you call in that function after calling `sync.Canceler()` will see the _new_ `sync.Done`.  `sync.Done` is a context var that is closed when the current context is to be considered canceled.  Watch it in your `select {}` loops, as you would have watched `ctx.Done()`, and check `sync.Err()` as if it was `ctx.Err()`: non–`nil` means the context has been canceled (or a group context failed).

#### `context func sync.GroupCanceler()`

Group Cancellation (example to replace `x/sync/errgroup.Group`):

   groupStopper := sync.GroupCanceler()

   sync.Group.Go(func() error { return someFunc() })

After calling `sync.GroupCanceler()`, the `sync.Done <-chan none` and `sync.Err error` variables are set as with `sync.Canceler`, except that `sync.Err` may also be `sync.GroupCancelled`.

You may have noticed that the previous example includes a closure that may close over its context vars, but it doesn't matter: they are the same as it is immediately called.  Any case which immediately calls a closure and does not return it has the same property.

`context var sync.Group` is a `sync.ErrGroup` and is implicitly set by `sync.GroupCanceler()`.  It will watch the goroutines it runs and if any of them return an error (or panic), then the rest of them get cancelled via `sync.Done` and `sync.Err`.

#### `context func sync.Deadline(when time.Time)`

* calls `sync.Canceler()` to make a new `sync.Err()` and `sync.Done`
* schedules a timer task to call the canceler
* also sets `time.Deadline`, a `time.Time` context var

#### `time.TTL() time.Duration`

Same as `time.Deadline.Sub(time.Now())`, although zero when expired

# Language Spec Changes

## Context Variables

A variable could be marked as "context" such as with `context var Context Partial` (example syntax for illustration purposes).  Unlike a regular variable declaration via `var`, the variable is initialized if it has been initialized anywhere in the call stack.  If no such contexts exist, then the variable is considered to have the zero value for the declared type.

Assignments to a context variable will have the effect of setting or changing the value in the current scope and any child scopes.  The change will be visible in any child scope which inherits the new context.

### Context Variable visibility

Context variables would have slightly different visibility rules, which should hopefully follow from what they are.

#### Cross–package context

The appearance _anywhere_ in a package, top–level or in a function, defines the _exported_ context variable `Context`:

    context var Context Partial

Say the package is called `logging`.  Other packages that import `logging` could then refer to this variable as `logging.Context`.  They could also set the value, presumably in this case with a partial log line, containing relevant IDs that are to be added to all scopes below (example: transaction IDs, user IDs).

Assigning to the variable takes on the new syntax (illustrative):

    context var logging.Context = logging.Context.With("userID", userID)

In this case, the value on the right sees the old value of the `logging.Context` variable, the value prior to the assignment.  In this usage, `logging.Partial` must be a pointer type, with a `With` method which allows a `nil` receiver and returns a modified `logging.Partial`, for this to safely "bootstrap" without requiring setup code.

#### Private context

Private context variables use a lowercase first letter of their name.

    context var currentUser iam.User

In this case, the variable name `currentUser` can only be accessed and set within the package the declaration is in, but it retains the dynamic scope regardless of where in the package it is defined.

## Context funcs

In addition to the 'var' keyword, the 'func' keyword can also be prepended with 'context'.  This declares a function or a method, called as normal, but with the modification that any changes the function makes to context variables are updated in the caller's context on return to that function.

This can be used to make helper functions that set up several context variables at once, without having to write lots of boilerplate/repeated code in your functions.  It can also be used for writing backwards compatible APIs, where you are migrating from a `package.WithContext(context.Context)`–style API to `context var` variables.

### Defining "child context"

In some cases, it is necessary to clarify exactly where the context "inherits" from.  For simple function calls to named functions and methods, as well as new goroutines, there is only one caller context from which scope can search, so that is used.

However, there are two cases where the "parent context" is potentially ambiguous: closures and bound method calls.  In order to provide the most flexibility when working with context variables, and cause the least possible surprise, the rule is: bound method calls _do not_ capture the context in which the method is bound with the instance, while closures _do_.

This means that the only context for a bound method call is the instance that is bound, and all dynamic scoped/context variables inside that method call are taken from the context that the bound method func is eventually called from.

By contrast, a closure by convention closes over everything by default: all variables in scope, which includes the dynamically scoped variables.

Where useful, to break ties context should be thought of as a read–only, append–only mutable state map, keyed by package and symbol, referenced by a pointer which is stored in a virtual register with a callee–save calling convention.  As may be clear, `context func`'s simply do not restore the context register.

### Standard Library Changes

In general it is proposed to change the context methods `Deadline()`, `Err()` and `Done()` to `context var` variables/callables in the `sync.` or `time.` namespace(s).  The hope is that this will in general be simpler and fewer lines of boilerplate to make work.

For backwards compatibility's sake, standard library functions that relate to context can be marked `context` to indicate they do not restore the caller's context on return.  The caller sees no difference to the function signature.  This can be used by new API calls to maintain a `context.Context`, and by old API calls to set the new API's variableZs.

While not a part of the language, seeing the issues that the stdlib functions have with context changes will help explore the problem space of moving from `context.Context` to less monolithic state.

### `context` library

The `context.Context` methods `Deadline()`, `Done()` and `Err()` are replaced by `context var`'s `time.Deadline`, `sync.Done` and `sync.Err()`, respectively.  Depending on the exact syntax used, `context.Context` can pass through to the `context var` values instead of explicitly saving them.

`context.WithCancel`, for instance, would be a `context func` method, and redefines the `sync.Done` (and `sync.Err`) context var inside, wraps the passed–in context and returns a cancellation callback.  This would allow a smooth upgrade path, and code which is ready can use the new API (described below) while the old API works with the new one.

Of course, the `context` package functions like `WithCancel`, `WithDeadline` and `WithTimeout` still return a `context.Context` instance, which does nothing especially magic or different from before, for backwards compatibility.

These compatibility functions are declared as `context func ...` and both set the new variable name and an implicit `context var` for the context; this cannot be read directly, but when you use `context.TODO()`, it is returned.  This means in your new code, you can use `context var` and then switch back to libraries that require `context.Context` using `context.TODO()` as the first argument.

### `sync` library

#### `context func sync.Canceler()`

For dynamic scope/context–based cancellation, call `sync.Canceler()`, which is called by `context.WithCancel`

    canceler := sync.Canceler()

This function as a side effect sets `sync.Done` and `sync.Err` in the calling scope's dynamic context.  Any function you call in that function after calling `sync.Canceler()` will see the _new_ `sync.Done`.  `sync.Done` is a context var that is closed when the current context is to be considered canceled.  Watch it in your `select {}` loops, as you would have watched `ctx.Done()`, and check `sync.Err()` as if it was `ctx.Err()`: non–`nil` means the context has been canceled (or a group context failed).

#### `context func sync.GroupCanceler()`

Group Cancellation (example to replace `x/sync/errgroup.Group`):

   groupStopper := sync.GroupCanceler()

   sync.Group.Go(func() error { return someFunc() })

After calling `sync.GroupCanceler()`, the `sync.Done <-chan none` and `sync.Err error` variables are set as with `sync.Canceler`, except that `sync.Err` may also be `sync.GroupCancelled`.

You may have noticed that the previous example includes a closure that may close over its context vars, but it doesn't matter: they are the same as it is immediately called.  Any case which immediately calls a closure and does not return it has the same property.

`context var sync.Group` is a `sync.ErrGroup` and is implicitly set by `sync.GroupCanceler()`.  It will watch the goroutines it runs and if any of them return an error (or panic), then the rest of them get cancelled via `sync.Done` and `sync.Err`.

#### `context func sync.Deadline(when time.Time)`

* calls `sync.Canceler()` to make a new `sync.Err()` and `sync.Done`
* schedules a timer task to call the canceler
* also sets `time.Deadline`, a `time.Time` context var

#### `time.TTL() time.Duration`

Same as `time.Deadline.Sub(time.Now())`, although zero when expired

# Informal Change

> Please also describe the change informally, as in a class teaching Go.

There is another type of variable in Go: a "context" variable.
This is a variable that is changed mainly for the benefit of ...

# Is this change backward compatible?

> Breaking the Go 1 compatibility guarantee is a large cost and requires a large benefit.

Yes, it's backwards compatible.

The main challenge will be making a solution which can gradually move people from
`context.Context` to the new context variable system.

Hopefully, the '`context func`' approach should work well.

# Orthogonality

> How does this change interact or overlap with existing features?

vs `context.Context`:

* Context is passed automatically, drastically reducing the need for refactoring

* Context variables are type safe

* Context variables are guaranteed to be passed down the stack
  (not dependent on intermediate contexts accidentally passing
  `context.TODO()` or `context.Background()`.

# Goals

> Is the goal of this change a performance improvement? If so, what quantifiable improvement should we expect? How would we measure it?

No, the main reason for this is avoiding the refactoring needs,
but also to make context, the category of which does not exist current.

> Would this change make Go easier or harder to learn, and why?

It's another thing that would have to be learned, but it should be weighed against the overhead that `ctx context.Context` imposes.  "Oh yeah, just pass that to every function" is not very satisfying statement to give to a new developer.

# Cost Description

> What is the cost of this proposal? (Every language change has a cost)

# Changes to Go ToolChain

> How many tools (such as vet, gopls, gofmt, goimports, etc.) would be affected?

# Performance Costs

> What is the compile time cost?

I have confirmed that the `context` keyword can be added to the language with minimal impact on compiling performance.

> What is the run time cost?

Negligable.  Some stack cost to context variables that would have been the case anyway (just at a different place).
Dual contet.Context / context var compatibility will double the overhead of Context.

There is some question about the runtime performance of `context.Context`.  It is certainly not an efficient
implementation with a lot of variables defined.  Both positive lookups to variables near the end of the context linked
list, and negative lookups, suffer from O(N) performance.  Worse, this is a nightmare for cachelines, with each level in
O(N) likely being a cache miss.

However, it is certainly possible for more efficient `context.Context` implementations to exist.  For instance, a Context
at any level could determine that it is running "hot" and build an authorative `map[any]any` of values.  This would require
that context be stricter about the type of context values it wraps, or have extra methods to handle the co-ordination of
keeping statistics and iterating over every value in the whole linked list.  Once built, the map would reduce positive and
negative lookups to O(1).  These maps can be built in a concurrent goroutine to the main program execution and seamlessly
swapped in once built.  There will be some finesse and tuning around when to build the maps.

If the use of `context var` variables grows as a result of the new language feature, then performance would be important.
This might not indicate a misuse of the feature; it might be that there are many use cases that have a good reason to use
context variables but don't, because of the overhead of passing `context.Context` values around.  Being able to depend on
availability of context variables could allow for some quite powerful libraries around instrumentation, logging, and
authentication.

# Prototype

> Can you describe a possible implementation?

Of course, `context.Context` forms a semantic prototype of the feature.

In order to prototype the type safety, a different syntax for getting values out could be used, following the pattern of
encoding/json and similar modules:

    var contextVariable someType
    ctx.Read("contextVariable", &contextVariable)

And for setting:

    ctx = context.Assign("contextVariable", contextVariable)

In this prototype, it would gather the caller's fully qualified package name, and confirm that the fully qualified package
name matches the caller's if the variable name is lower case.  Internally, variables would be tracked with the type:

    type packageKey struct {
        packageName string
        symbolName  string
    }

    type contextValueMap map[packageKey]any

Otherwise, the module would function quite similarly.  It would not provide the various random features built into
`context.Context`, like `Done()`/`Err()` and `Deadline()`; those would be provided by external packages that would emulate
the `sync.` and `time.` parts described above.  The core `context.Context` module can be brought out into a package and
implemented using the prototype module, to show what changes would have to occur, and then programs being tested for
backwards compatibility would just change their `import` statements to import the prototype adjusted standard library
package.
