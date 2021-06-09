# Scrap your parentheses: new operator for providing arguments

In most of modern programming languages parentheses are used as a function application operator to its arguments: 

```python
q (x, y z)
```

But in Haskell function application operator is space character:

```haskell
q :: a -> b -> c -> d
x :: a
y :: b
z :: y

q x y z
```

Buy wait a minute! We apply function `g` to some arguments, but there are more space characters than one. Doesn't it mean that we have more than one function application? Well, yes:

```haskell
(((q x) y) z)
```

We actually get several function appliactions in curreid form:

```haskell
q x :: b -> c -> d
q x y :: c -> d
q x y z :: d
```

Does that mean that we can remove parentheses? No, because we can get function applications as arguments, we could be too lazy to think about new names for intermediate expressions:

```haskell
q :: a -> b -> c -> d
p :: b -> c

q x y (p y)
```

And here is the case we can't remove parentheses because otherwise typechecker would suggest we provide `q` two different arguments:

```haskell
q :: a -> b -> (b -> c) -> b -> ???

q x y p y
```

Fortunately, we already have a special infix operator in `base` that can help us remove parentheses:

```haskell
q :: a -> b -> c -> d
p :: b -> c

q x y $ p y
```

This operator (`$`) is called application which just explicitly separates function and its argument:

One function is definetely not enough and as soon as we tried to use function applications, we want just to write:

```haskell
($) :: a -> (a -> b) -> b
o :: d -> e

o (q x y $ p y) === o $ q x y $ p y
```

Looks good! But what if instead of providing arguments explicitly we could just somehow combine functions together without mentioning any arguments?

![...](https://hsto.org/getpro/habr/upload_files/198/bb3/99a/198bb399acc2fe0dd0b978ed2aa99914.jpeg)
> "What is a Category? Definition and Examples" (c) Math3ma"

Composition is the essence of category theory so we have a method to compose functions in associative chain. Such a notation is often called pointfree (despite the fact that there are more points):

```haskell
f :: a -> b
g :: b -> c
g . f :: a -> c

(.) :: (b -> c) -> (a -> b) -> (a -> c)

g (f x) === g . f
```

A choice between composition and application is often a question about stylistic preferences because they are structurally equal:

```haskell
g . f === g $ f x
```

If we have a free variable we can provide it to a function and then use compositions - I get used to write code like this because it looks nicer in my opinion:

```haskell
j $ h $ f $ g $ x === j . h . f . g $ x
```

But there is a problem with these two operators - they can consume only one argument. If we prodive some expression as arguments we have to use parentheses:

```haskell
f :: a -> b -> c
g :: a -> b -> c -> d
h :: c -> d -> e

h (f x y) (g x y z)
```

We can remove parentheses with application only on the right side. 

```haskell
h (f x y) (g x y z) === h (f x y) $ g x y z
```

Can we do the same on the left? To understand where the problem actuall is, let's analyze composition (`.`) and application (`$`) a bit. Both of these operators are right associative. Associativity is all about parentheses. Right associativity means that we group parentheses to the right: 

```haskell
f . g . h === f $ g $ h x === f (g (h x))
```

Functions in Haskell are already in curried form so we don't have to provide arguments all at once, we can provide them one by one: 

```haskell
f :: a -> b -> c -> d
f x :: b -> c -> d
f x y :: c -> d
f x y z :: d
```

So if we want to introduce a new operator that could consume several arguments as any expressions we need to make it left associative so that it could consume arguments from right to left:

```haskell
(???) :: (a -> b -> c -> ...) -> a -> b -> c -> ...
((??? x) y) z) ...
```

Let's take some known function from `base` to test our new operator, `maybe` for example:

```haskell
maybe :: b -> (a -> b) -> Maybe a -> b
```

Now let's choose some ASCII symbol as name for our new operator that seems to be not widely used.

```haskell
(#) :: (a -> b -> c -> ...) -> a -> b -> c -> ...
f # x y z ... = ???
```

Aside associativity we need to pick precedence - it's a number from 0 to 9 to define priority. The higher the number, the lower the priority.

```haskell
infixr 9 .
infixr 0 $
```

That's why compiler group parentheses around `$` but not around `.`: 

```haskell
h . g . f $ x === (h . (g . f)) $ (x)
```

So we can choose any number from 0 to 9. Let's pick something in between:

```haskell
infixl 5 #
```

But how could we define it? Actually it's an application operator in the essence but focused on function and not on arguments:

```haskell
f # x = f x
```

So... how it's gonna work? Let's take an example with `maybe`. Since we have functions in curried form we can provide arguments one by one:

```haskell
maybe :: b -> (a -> b) -> Maybe a -> b
maybe x :: (a -> b) -> Maybe a -> b
maybe x f :: Maybe a -> b
maybe x f ma :: b
```

Currying means that we can group parentheses to the left! 

```haskell
maybe # x # f # ma === ((maybe # x) # f) # ma

maybe # "undefined" # show . even # Just 1 === "False"
maybe # "undefined" # show . even # Just 2 === "True"
maybe # "undefined" # show . even # Nothing === "undefined"
```

If provided arguments are too big and we don't want to think about names we could indent provided arguments as a block with preceded `#` operator:

```haskell
string_or_int :: Either String Int
either :: (a -> c) -> (b -> c) -> Either a b -> c

either 
	 # print . ("String: " <>) 
	 # print . ("Int: " <>) . show
	 # string_or_int   
```

I haven't seen such an operator before but I'd like to use it if `.` and `$` can't manage parentheses well. Leave your comment (I think it's gonna be on Reddit) if something similar exists but by some reason is not widely used.
