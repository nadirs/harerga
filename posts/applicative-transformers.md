---
title: Applicative Transformers: IdentityT
tags: haskell, applicative, transformers
date: 2014-06-28
---

Recently on Twitter, after reading about someone using Monad transformers for their first time, I've written some, as random as usual, tweet about the possibility of having applicative transformers.

Promptly [\@bitemyapp](http://twitter.com/bitemyapp) gave me some food for thoughts. He suggested to start simple, by trying to rewrite an Applicative version of `IdentityT`, which is the identity tranformer for monads.

<blockquote class="twitter-tweet" lang="en"><p>.<a href="https://twitter.com/nadirsampaoli">@nadirsampaoli</a> a monad transformer takes a monad and returns a monad. Start simple - can you make IdentityT applicative?</p>&mdash; FKA da bear ʕノ•ᴥ•ʔノ (@bitemyapp) <a href="https://twitter.com/bitemyapp/statuses/482643698727915520">June 27, 2014</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## The initial questions

Everything started with some questions thrown a bit carelessly:

 1) Do Applicative transformers exist?
 2) Would they be useful?
 3) Does it even make sense to think about such a thing?

With some new (for me!) concepts to learn about, I entered the cabal sandbox I use for experiments, fired up a vim buffer, and started rewriting IdentityT as an Applicative Functor, instead of a Monad.

## What exactly is IdentityT?
First of all, I needed to figure out what exactly `IdentityT` was about. A quick Hoogle search brings up the Hackage page for the package named [Control.Monad.Trans.Identity](http://hackage.haskell.org/package/transformers-0.4.1.0/docs/Control-Monad-Trans-Identity.html#t:IdentityT).

Without reflecting too much, I started writing down some mindless implementation of IdentityT as a Functor and Applicative instance:

```haskell
{-# OPTIONS_GHC -Wall #-}
{-# LANGUAGE KindSignatures #-}

module ApplicativeTransformers
    ( IdentityT(..)
    ) where

import Control.Applicative

newtype IdentityT (f :: * -> *) a = IdentityT { runIdentityT :: f a } deriving (Show)

instance Functor f => Functor (IdentityT f) where
    fmap h (IdentityT f) = IdentityT $ fmap h f

instance Applicative f => Applicative (IdentityT f) where
    pure = IdentityT . pure
    (IdentityT h) <*> (IdentityT f) = IdentityT $ h <*> f
```

**Note:**  
GHC extension `-XKindSignatures` lets you constrain *IdentityT*'s type parameter `f` to that kind (i.e. the kind required to create Functor and Applicative instances).

Now, this was just an exercise to ensure I understood what I was actually putting together (keep reading, it's going to be no timebefore I realize I didn't really understand what's going on). GHC has another extension, named `-XGeneralizedNewtypeDeriving` that allows you to avoid the *chore* of writing by hand type class instances that could be inferred by the compiler. So the previous snippet becomes:

```haskell
{-# OPTIONS_GHC -Wall #-}
{-# LANGUAGE KindSignatures, GeneralizedNewtypeDeriving #-}

module ApplicativeTransformers
    ( IdentityT(..)
    ) where

import Control.Applicative

newtype IdentityT (f :: * -> *) a = IdentityT { runIdentityT :: f a }
    deriving (Show, Functor, Applicative)
```

## Implementing the actual transformer

Ok, now with that in place, the next thing to do would be to write a transforming function that operates on `IdentityT`. What's its type signature?

```haskell
mapIdentityT :: (f a -> g b) -> IdentityT f a -> IdentityT g b
mapIdentityT h x = undefined
```

I couldn't find a way to apply `h` to `x` using just Applicative functions: the problem is that `fmap`, `pure`, etc. don't operate on IdentityT but on the type it wraps. I thought it should have been possible to just do something like:

```haskell
mapIdentityT = fmap
```

<sub>
Here, by the way, is where I realized the Applicative instance for IdentityT refers directly to the type parameter of the applicative functor wrapped by the newtype.
</sub>

Since the `IdentityT f a` implementation of `fmap` expects a function `a -> b` instead of `f a -> g b` I couldn't use it. What I came up with wasn't anything better than unwrapping and re-wrapping the IdentityT instance and just apply the function to the applicative that lives inside the newtype wrapper:

```haskell
mapIdentityT :: (f a -> g b) -> IdentityT f a -> IdentityT g b
mapIdentityT h = IdentityT . h . runIdentityT
```

There is apparently really not much difference between a monadic and a "just" applicative implementation of the **identity transformer**. This somehow answers the first of those three question I asked initially or, at least stands for the Identity transformer.

## Can I use it?

Let's see what we can do with this applicative-only IdentityT.

```haskell
> let foo = IdentityT (Just 5)
> foo
IdentityT {runIdentityT = Just 5}
```

Ok, now we need a function `f a -> g b` (let's call it `bar` for the sake of meaningful names), which in this case would be `Num a => Maybe a -> g b`. Lets say we want a list of strings, so the signature ends up being:

```haskell
bar :: (Num a, Show a) => Maybe a -> [String]
```

Its implementation is as follows:

```haskell
> let bar = maybe [] (pure . show)
```

Finally we map `bar` to `foo`:

```haskell
> mapIdentityT bar foo
IdentityT {runIdentityT = ["5"]}
```

At least it works, although I'm not sure this answers the second of my questions (*Is it useful?*).

### Wrap up and next steps

Next thing to do is figuring out how to implement more complex transformers and probably learning about Yoneda and Coyoneda, whatever they are.

There's something that does not feel right with this exploration, I'm probably missing something (part of the problem has to do with the fact I haven't actually ever used monad transformers). So, feel free to let me know about nonsense I might have written here; I'm on Twitter [\@nadirsampaoli](https://twitter.com/nadirsampaoli).

---

<sub>
The source code for the thing I called ApplicativeTransformers is on Github: [**nadirs/applicative-transformers**](https://github.com/nadirs/applicative-transformers/)
</sub>
