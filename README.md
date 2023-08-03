# Python style guide


## Before we start, some values

- Immutable > Mutable
- Explicit > Implicit
- Composition > Nesting
- Use state as little as possible
- Re-use code as much as possible

## Immutable > Mutable 

- Use `tuple` instead of `list`
- Use `frozenset` instead of `set`
- Use `frozendict` when you need to serialize something

## Be concise

Motivations:

- Bug occurrence is correlated with amount of code. The more code you write, the more bugs you create.
- Long implementations are usually a sign for duplication or design problems.
- Concision mostly means more code reuse, which is good for multiple reasons.

So - write less code, delete code often.

## More signal, less noise

Put your core logic at the center, and remove "fluff" code that might confuse the reader.

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

## Composition > nesting

We prefer compose_left(f, g) over calling g inside f.
Why?
1. It's easier to understand from the "outside" what the composition is doing (first g, then f)
2. It's easier to reuse g in a different place, or to refactor elements from f.

No:

```python

def f(x):
   result = g(x)
   return do_something(result)

f(x)
```

Yes:

```python
gamla.pipe(x, g, do_something)
```

## Don't introduce meaningless parameters

`f(x)` and `f` mean the same thing. Always use `f` and not `f(x)`.

No:

```python
(x for x in people if my_filter(x))
```

`x` is a meaningless name.

No:

```python
(person for person in people if my_filter(x))
```

Even if we had written `person` - we are just creating redundant noise.

Yes:

```python
filter(my_filter, people)

```

### Use compose_left instead of pipe when possible

When you have a pipeline that needs an input from the top, you don't need a def, you can use a composition.

This reduces the amount of noise, and doesn't require you use ```await``` in async functions. 

No:

```python

def my_pipeline(x):
    return gamla.pipe(x, f, g)
```

Yes:

```python
my_pipeline = gamla.compose_left(f, g)

```


### Use 'if' for values, ternary for functions 

If you have a condition that's not dependant on input, use ```if```
When you see `when`/`unless`/`ternary`+`always`/`just`, it means we're building the pipeline while running it, which can be confusing when read. If you have a constant value which determines how the pipeline should work, you should use an ```if``` to condition on it.

No:

```python
my_function = gamla.ternary(gamla.just(my_var), f, g)
```

Yes:

```python
my_function = f if my_var else g
```

## Use currying instead of default values

Python's default values makes it hard to understand function calls, especially when used with currying. 

No:

```python
@gamla.curry
def do_something(x, y=None):
    if y is not None: # Hmm apparently `y` can be `None`. Let's result to defensive programming.
        ...

# This function can be called in many different ways:
do_something(x) # Means use `y` as `None`...
do_something(x) # Means return a function that can gets `y` (if it's curried)

# These are all the same:
do_something(y=y, x=x)
do_something(x=x, y=y)
do_something(x, y)
```

Default values make refactoring difficult, because not all call sites look the same.
Moreover, default arguments "help" developers create signatures with many arguments, which mean more clutter and less readability.


When you have many callers needing some default behaviour and one caller that needs a custom argument, make two functions and use currying to avoid signature duplication.

No:

```python
def call_openai(prompt, max_tokens = 64):
    return openai.Completion.create(
        model="text-davinci-003", prompt=prompt, max_tokens=max_tokens
    )

```

Yes:

```python
@gamla.curry
def call_openai_with_custom_max_tokens(max_tokens, prompt):
    return openai.Completion.create(
        model="text-davinci-003", prompt=prompt, max_tokens=max_tokens
    )

call_openai = call_openai_with_custom_max_tokens(64)

```

## Naming

### File names
If you have a directory dealing with something, and internal files with specifics for that something, do not repeat the directory name in the internal filenames.

No:

`fruit/fruit_apple.py`
`fruit/fruit_tomato.py`

Yes:

`fruit/apple.py`
`fruit/tomato.py`

Avoid ```utils``` files, as it is a lazy naming convention, and becomes meaningless really fast, creating non-trivial dependencies across the project.

### Functions

Make sure every word in the function name is meaningful. Therefore, words like ```get```, ```run```, ```process```, ```make```, ```handle```, ```do``` can most times be removed or replaced with a more specific verb like ```fetch```, ```read```, ```load``` etc.

No:

`def get_urls_related_to_text(text)`

`def get_appoinments_from_api(endpoint_url)`

`def run_action_trasnformation(action)`


Yes:

`def urls_related_to_text(text)`

`def fetch_appointments(endpoint_url)`

`def action_to_sentence(action)`

### To name or not to name

Factor out and name functions when one of these conditions apply:

1. When the function are too long and clutter the idea of the surrounding context.
1. When you want to avoid repeating yourself.

Avoid naming functions when the name is of similar length to the implementation.

No:

`filter_larger_than_1 = gamla.filter(operator.lt(1))`


## Functions should do one thing

Large and sophisticated blocks of code should be under scrutiny:

- Is there actually more than one thing going on here?
- Can I give a good name for a subset of this logic?
- Is the order arbitrary?

If the answer to one of these is yes, then it's time to factor out, and use composition instead of nesting.

## No dead code

Remove all code which is not used in the application. This includes "preparing for the future" or "this might be useful again" patterns (odds of correctly predicting what will be needed or not are slim).


## Some more small things

- Privatize as many functions as possible, using the underscore, `_like_so`.
- Add typing on every function. It greatly helps readability and code safety.
- Always import the modules, not the inner names. e.g. don't do: `from my_module import MyClass`, instead do: `import my_module`. There is one exception to this rule which is the `typing` module.
