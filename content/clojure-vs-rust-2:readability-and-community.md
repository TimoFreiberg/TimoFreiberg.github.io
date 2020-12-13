+++
title = "Clojure vs Rust 2"
date = 2020-12-13
draft = true
+++


## Readability

TODO remove this section, maybe move it into a followup article?

In Clojure, I designed my own little ad-hoc datatype patterns, but it was always hard to remember how they were shaped when I had to do something with one I hadn't touched in a while. (TODO needs example, maybe my error/log map?)
The most sensible way to fix this would have been to introduce functions for this new "data type" that do everything i want to do with the type and never directly access the internals in any other place - exactly what Java/Rust are forcing me to do (that is, if the fields are private).
Now I'm aware I'm admitting to a lack of personal discipline, but it's hard for me to future-proof my code when I can actually see in the REPL what the structure is, so I can just write code that directly uses it.

In Rust on the other hand, this problem mostly goes away.
It's just idiomatic to introduce types and then implement the things I want to do with them as methods on those types instead of freestanding functions somewhere else.
Calling methods even looks nicer, so I am extra encouraged to write the code that uses the internals of a type near the definition of the type.
This is only one of many details where Rust successfully designed a Pit of Success (https://blog.codinghorror.com/falling-into-the-pit-of-success/).

Now, so far I've just been describing a very normal "slightly above trivial codebase + human memory = some design required to make it readable" situation.

There was another reason why I was very happy to have a static type system again, and it was caused by the complexity of the data I was handling, which I could no longer fit in my head at once.
See, each of the entries I was diffing had one field which was always a conjunctive normal form (https://en.wikipedia.org/wiki/Conjunctive_normal_form).
What I had to do with that formula was replace each term with a Disjunction of other entries (this formula specified dependencies between entries, you see. I was not too happy with that design).

The resulting data type was therefore a conjunction of disjunctions of terms (meaning this is negatable) of disjunctions of other rules.
The term layer in there unfortunately prevented the easy flattening of two levels of Disjunctions.

Now this might have just been me, but I was struggling to keep the current level I was operating on in my head while I was actually writing the code.
Reading the code a few days later was even worse.
Writing tests obviously helped, but it didn't make the problem go away.
You can probably imagine that the code to process what is essentially several levels of nested lists consists of what is essentially nested loops.
If you make a mistake and go one level too deep, you might loop over the leafs (which might give you a list of the key-value pairs of the rule object if it was a hashmap or a list of characters if it was a string).

In Rust, the whole thing was a bit better.
I was still regularly completely out of my depth, but the reason I could even progress was that I had a machine checking whether I was really passing a `Conjunction<Disjunction<Term<Disjunction<T>>>>` to that function.

TODO ranty? not sure. need to edit
