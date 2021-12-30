# Code writing guidelines

## Code reviews

Code reviews are a great tool to enhance code quality and stability. It works like this:

1. The author opens a merge request and assign a reviewer.
1. The reviewer writes some comments and assign the review back to the author.
1. The author resolves all comments (by fixing or replying).
1. This goes on in iterations until the reviewer is satisfied. Once this is the case they will click on the "Approve" button or write "LGTM".
1. At this point the author can merge their request (pending on resolving all open issues).
1. Authors should always squash their commits when merging, as it makes things easier to understand when reading through the commit history. If **all** their individual commits make sense and have valid descriptions, they can merge without squashing.

The reviewer is responsible for keeping the overall correctness and code health level of the entire system. In addition (and without contradiction), the reviewer is responsible for helping the author merge their code as quickly as possible, allowing for future iterations to happen in distinct concentrated efforts. Unmerged code is a form of technical debt and should be eradicated as quickly as possible.

Most of the time these goals do not clash. But there are some gray areas.

A good rule of thumb is that code health should never decrease, not even locally. If a commit constitutes a regression, it is the reviewer responsibility to comment and require a fix before merge, or adding a `TODO` if time is of the essence and the author wishes to push code and return to the problem offline (but immediately after). On the other hand, it is perfectly fine to push code without changing code health for better or worse.

Reviewers should be wary of overextending their duties. For example, reviewers should avoid expanding the limits of the project as the author sees it, and must always look for the minimal stopping point that adheres to increasing/not-changing code health, while keeping correctness and increasing usefulness.

Avoid:

- "Oh you fixed this, then please also fix that" where "that" refers to untouched lines of code which are not directly related to the project.
- "It could be cool if now everywhere in the code we use this new pattern that you invented, please fix everywhere".
- "Since you have written a test for this function, maybe you can cover all others as well?"

It is fine to have these comments but stress that they are optional (e.g. "consider ...").

## Consistency helps readability

Inconsistencies cause distraction while reading code. In reviews, they waste both reviewer and author time and effort, while they fix them. At times, people argue about the best convention.

To side step this issue entirely, autoformat everything (including configurations and markdown files).

Prefer opinionated autoformatters which rewrite the code from its abstract syntax tree. Automatic consistency implies removing redundnat degrees of freedom.

## Code comments

- Comments, as all text which is read more than once, should be well written, without typos with right capitalization and punctuation.
- Do not write comments that just describe the implementation, that's what the code is for.
- Use comments as a last resort to illuminate something a reader might not immediately understand.
- There should be very little comments in general, most code should be self explanatory. In case it's not, prefer refactoring it to commenting.

## Be concise

Motivations:

- Bug occurrence is correlated to amount of code. The more code you write, the more debugging has to be done.
- Long implementations are usually a sign for duplication or design problems, and duplication is the opposite of reuse.
- Code reuse enables easier code optimization (many usages are improved by a single optimization).
- Scope size correlates to amount of possible interactions (in a scope, the reader must assume that any two elements can interact).

Therefore perhaps the only objective metric by which a design can be examined is code length.

So - write less code, delete code often.

## Optimize signal to noise ratio

Some forms of duplication are easy to spot, simply by the noisy feeling you get reading them.

No:

```python
f = gamla.remove(
    gamla.anyjuxt(
        gamla.equals("hello"),
        gamla.equals("hi"),
        gamla.equals("what's up"),
        gamla.equals("good afternoon"),
        gamla.equals("morning"),
        gamla.equals("good evening"),
    )
)
```

Yes:

```python
f = gamla.remove(
    gamla.contains(
        {
            "hello",
            "hi",
            "what's up",
            "good afternoon",
            "morning",
            "good evening",
        }
    )
)
```

## Flat is better than nested

Humans are not so good at counting brackets (as some linguists might believe). Help them avoid it by flattening nested structures.

No:

```python
result = tuple(map(increment, [1,2,3])))
```

Yes:

```python
result = gamla.pipe(
    [1,2,3],
    gamla.map(increment),
    tuple
)
```

## Work on the single case

Try and minimize transitions between the multiple values space and the single value space.

No:

```python
result = compose_left(map(f1), map(f2), map(f3))
```

Yes:

```python
result = map(compose_left(f1, f2, f3))
```

This will simplify the code, and remove cognitive overhead when writing it, as it's easier to hold in mind one object than a plurality.

## Work in the right level of abstraction

Point free (aka tacit) programming is a way to avoid naming-noise and signature duplication.

Some names don't really add value or readability, but we have to write them because we decided to work in the value level, instead of the function level. When this happens, it is a sign of working in the wrong level of abstraction.

No:

```python
(x for x in people if my_filter(x))
```

`x` is a meaningless name.

No:

```python
(person for person in people if my_filter(x))
```

Even if we would have written `person` - we are just creating more visual noise, by repeating this information 3 times (twice for `person` and another time in `people`).

Yes:

```python
filter(my_filter, people)

```

Here we imply the idea cleanly with considerably less noise.

### Eta reduction

One technique to move from working in the value level to the function level is to remove variables which appear on the ends of the implementation. This is known as "eta reduction".

For example, when you have a pipeline which needs an input from the top, you don't need a function.

No:

```python

def my_pipeline(x):
    return gamla.pipe(x, f, g)
```

Yes:

```python
my_pipeline = gamla.compose_left(f, g)

```

Another example is when you use some of the input on top and others in the bottom. In these cases you can simplify the signature by returning a function. Then you can avoid redundant currying and arguments by making point full curried functions into point free uncurried higher order functions.

No:

```python
@gamla.curry  # Redundant currying.
def my_func(x, y, z):  # Redundant argument, duplicated with the signature of `run`.
    return gamla.pipe(z, run(x), run_also(y))  # Redundant pipe stage.

gamla.pipe(
    ...
    my_func(x, y),  # Getting `z` through the pipe.
)
```

Yes:

```python
def my_func(x, y):  # Simplified signature, removed currying.
    return gamla.compose_left(run(x), run_also(y))  # Removed one stage.

gamla.pipe(
    ...
    my_func(x, y)  # Applying the resulting function on the `pipe` input without naming.
)
```

### Don't mix pipeline construction and pipeline running

One example for this is overusing functional conditionals with constant values.

When you see `when`/`unless`/`ternary`+`always`/`just`, it means we're constructing the pipeline mid-running it, which is a kind of a level mixing. If you have a constant value which determines how the pipeline should work, it should usually be read and used prior to the running time of the pipeline. This is similar to factoring out values from a loop when it never changes.

No:

```python
my_function = gamla.ternary(gamla.just(my_var), f, g)
```

Yes:

```python
my_function = f if my_var else g
```

## Purity and immutability make life easier

- They make debugging, caching and serializing easier.
- Avoid global state.
- Do not use mutable classes - they are often just a bunch of hidden global variables.
- Minimize state even within a function (e.g. unnecessary variables), instead of naming something within the function scope, name its logic as a static function.
- Use pure functions (no side effects).

### Store facts, lazily compute derived attributes over them

Rich Hickey explains it well in his "The Value of Values" short talk: <https://www.youtube.com/watch?v=-6BsiVyC1kM>.

### No global caching

Mutable objects are hard to reason about, but when they're global it's even worse (what code changes them? when?).

Most developers know that. But most developers don't think of cache decoration as a global mutable thing, while it definitely is. Global caches make the code "sometimes" faster, who knows when and why. They make optimizing code much more difficult than it is, and require developers to reload the code often to get a "clean slate". They make writing latency tests especially hard and require mocking. They are also never garbage collected, which can lead to a nontrivial memory footprint which is hard to track.

For these reasons, avoid global caching altogether. Instead use local caching patterns, e.g. dictionaries or injected computation inside garbage collected scopes (functions).

No:

```python
import functools

@functools.lru_cache # I am a global waiting to cause harm!
def something_slow(x):
    ...

@gamla.curry
def something_that_is_called_a_lot(x, element):
    return something_slow(x) + element


def loop(x, elements):
    return tuple(map(something_that_is_called_a_lot(x), elements))
```

We don't want to combine the functions, but we also don't want to call `something_slow` more than we need to. We can factor outside to have our cake and eat it too.

Yes:

```python
import functools

def something_slow(x):
    ...

@gamla.curry
def something_that_is_called_a_lot(slow_result, element):
    return slow_result + element


def loop(x, elements):
    slow_result = something_slow(x)
    return tuple(map(something_that_is_called_a_lot(slow_result), elements))
```

## Avoid ambiguity

Programming languages often introduce features to allow for multiple ways to use a function.

The usual suspects here are optionality/nullability and default arguments.

No:

```python
@gamla.curry
def do_something(x, y=None):
    if y is not None: # Hmm apparently `y` can be `None`. Let's result to defensive programming.
        ...
```

This function can be called in many different ways:

```python

# Someone writing this may expect two different outcomes:
do_something(x) # I mean use `y` as `None`...
do_something(x) # I mean return a partially applied function that can get `y` as well...

# These are all the same:
do_something(y=y, x=x)
do_something(x=x, y=y)
do_something(x, y)

```

So suddenly default arguments and currying together can have surprising results.

These language features things, while making it easier to write code, make it harder to reason about it, or refactor it.

Default arguments are especially dangerous as they "help" developers create signatures with many arguments.

No:

```python

def pizza_spec(size, topping, part="entire-pizza", crust=None, beverage=None):
    ...
```

Defaults make it hard to reason about the code (when is this argument given? when is it not? do I have to give argument topping alongside `part`?).

This variation and implicit requirements make refactoring difficult, because not all call sites look the same.

When refactoring, it helps if it's very clear how a function is used, otherwise one would have to consider all possible cases, and this number grows exponentially with the variability of arguments, forcing the coder to go over all cases and divide them.

Consequently - never use default arguments. They are not conducive to healthy code.

To solve the common case where you have many callers needing some default behaviour and one caller that needs to configure something in a different way, make two functions and use currying to avoid signature duplication.

No:

```python
def calculate(x, addition, multiplication=1):
    .return (x + addition) * multiplication

```

Better:

```python
@gamla.curry
def calculate(multiplication, addition, x):  # Note the new variable ordering.
    ...

addition = calculate(1)  # Users not giving `multiplication` will use this function.
```

This way we know that users which do not give `multiplication`, and when refactoring we can address them as a separate group.

But really the best way is to just separate the function, and get smaller signatures.

Best:

```python
def multiplication(x, y):
    return x * y

def addition(x, y):
    return x + y

```

In pizza world, you can split the large function into smaller ones:

```python
def topping_request(topping, part):
    ...

combine_requirments = gamla.compose_left(gamla.mapcat(dict.items), dict)


order = combine_requirments(topping_spec('mushrooms', 'all'), size_spec('medium'))

```

If no crust information is needed, then it's just not there. It's also now obvious that `part` refers to the topping.

## Code should be context free as possible

For reuse and readability, a function's name or description should only refer to the code it has, not to the context where the function would be used.

No:

```python
def make_price_slightly_higher(price: int) -> int:
    return price + 1

```

- I don't really need to know all that to understand this function.
- Maybe I want to reuse this in another context.

Yes:

```python
def increment(n: int) -> int:
    return n + 1
```

Same goes for filenames. If you have a directory dealing with something, and internal files with specifics for that something, refrain from repeating the directory name in the internal filenames.

No:

`fruit/fruit_apple.py`
`fruit/fruit_tomato.py`

Yes:

`fruit/apple.py`
`fruit/tomato.py`

## Functions should do one thing

Large and sophisticated blocks of code should be under scrutiny:

- Is there actually more than one thing going on here?
- Can I give a good name for a subset of this logic?
- Is the order arbitrary?

If the answer to one of these is yes, then it's time to factor out, and use composition instead of nesting.

Here are some concrete examples.

### Self-disable patterns

If a function starts with some logic of when the core logic should occur, factor this logic out.

No:

```python
def f(a, b, c):
    if check(a):
        return
    ... # Rest of logic

f(a, b, c)
```

`f` is aware of `check`.

Yes:

```python
def f(a, b, c):
    ... # Rest of logic

if check(a):
    f(a, b, c)
```

`f` and `check` are oblivious to each other, becoming simpler.

## No dead code

Remove all code which is not connected to the application. This includes "preparing for the future" or "this might be useuful again" patterns (odds of correctly predicting what will be needed or not are slim).

Similarly, debugging functions belong in external libraries.

## To name or not to name

Factor out and name functions when one of these conditions apply:

1. When the function are too long and clutter the idea of the surrounding context.
1. When you want to avoid repeating yourself.

Avoid:

1. Naming functions where the name is of similar length to the implementation.

   No:

   ```python

   filter_larger_than_1 = gamla.filter(operator.lt(1))
   ```

1. Repeating the filename when only one target is exported.

   No:

   ```javascript
   const f = () => true;

   export default f;
   ```

   (In `f.js`)

   Yes:

   ```javascript
   export default () => true;
   ```

   (In `f.js`)

## Correct use of currying

Non-unary functions are commonly used in compositions using currying. To make this experience as nice as possible, it is important that the argument order matches the expected calling sequence.

When unsure about usage, a good rule of thumb is - the more fundamental to the function operation an argument is, the earlier it should be in the signature. This is why higher order functions like `map` and `filter` always have the function as their first parameter.

No:

```python
@gamla.curry
def foo(x, y):
    ...


gamla.compose_left(foo(y=2), bar)
```

Yes:

```python
@gamla.curry
def foo(y, x):  # Note the argument order has changed.
    ...


gamla.compose_left(foo(2), bar)
```

## Avoid "utils"

Utils is a lazy naming convention, and becomes meaningless really fast, creating non trivial dependencies across the project. If something is really a util it belongs in `gamla` or any other open source lib, not in a normal repo.

## Avoid redundant `pipe`s

No:

```python
gamla.pipe(x, f)
```

yes:

```python
f(x)
```

However if `f` is an expression with brackets, then a `pipe` can make it nicer:

No:

```python
expression_with_brackets(and_nesting)(x)
```

Yes:

```python
gamla.pipe(
    x,
    expression_with_brackets(and_nesting),
)
```

## Python specific stuff

- Use the `black` formatter without any configurations.
- Use the underscore convention to name things which are private to the module, `_like_so`.
- Unless you have a strict efficiency requirement, avoid the mutable data structures - `set`, `list`, etc'. Prefer their immutable alternatives - `frozenset`, `tuple`. In the case of `dict` there is no good alternative, so no other choice. This does not apply to unassigned values such as anonymous literals. E.g. it's fine to do `print([1,2,3])`, or return a mutable datatype within a pipeline that does not expose it.
- Don't use `defaultdict` - it encourages mutation patterns instead of creating a new `dict`.
- Take care to retain stability in ordered outputs, otherwise tests become flaky and replaying what happened becomes harder. If order has no meaning use `frozenset`.
- Use typing as much as you can, especially on APIs. It greatly helps readability and code safety. Avoid `Optional` and `Union` as they increase ambiguity.
- Note that `Optional` typing for a function argument does not imply it is a default variable. This is a common mistake. `Optional[x]` is equivalent to `Union[None, x]`.
- A tuple of a repeating type `X` is written like so: `Tuple[X, ...]` (note the `...`, this is different from other containers), while a typing for a tuple with one element would be `Tuple[X]`.
- Always import the modules, not the inner names. e.g. don't do: `from my_module import MyClass`, instead do: `import my_module`. There is one exception to this rule which is the `typing` module.
- Use `asyncio` rather than threads or processes, because of the GIL and other overheads. If you need multiprocessing, factor it out to a service.

## Javascript specific stuff

- Use the `prettier` formatter without any configurations.
- Use `const` to declare new variables. Use `let` if you must. Never use `var`.
- In `React`, do not use the `Component` class, instead use functional components.
- Use only arrow functions, and aspire to have them body-less, i.e. instead of `const f = x => {return x+1};` do `const f = x => x+1;` to get a better signal to noise ratio, especially when reading functional components.
- When possible prefer the `export default` to avoid causing a duplication between a filename and its contents.
- For safety, prefer keyworded args when a function is not unary. E.g. `const f = ({arg1, arg2, arg3}) => {...}` instead of `const f = (arg1, arg2, arg3) => {...}`.
