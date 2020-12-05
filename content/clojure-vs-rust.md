+++
title = "Clojure vs Rust"
date = 2020-12-02
+++


## Intro

I was quite into Clojure a while ago and was looking for something to practice on.
So I started writing something about some relatively complicated masterdata that some of my colleagues recently started to work with.

I started writing some parsing and validation logic and exploring the domain.
Inspired by some prod issues caused by accidental maintenance errors in this data
(which was maintained in Excel sheets [link to the excel fails during the corona crisis])
, I proposed writing a semantic differ to compare two versions of the excel sheet.
This turned out to be the most algorithmically complicated program I have written so far.

After getting the core diffing algorithm working, I regularly worked at improving its performance as early versions took many minutes to run.
I learned how to profile my code using https://github.com/clojure-goes-fast/clj-async-profiler which helped find inefficiently implemented hot loops.
I also redesigned the algorithm several times, allowing it to skip lots of work.
A few times, profiling helped me find small issues that made enormous differences (which were usually silly mistakes)
, but I quickly reached a point where improvements in the algorithm were the only way to realistically make the whole thing faster.

I only worked on the project on the side, so these frequent redesigns forced me to regularly relearn parts of the codebase.
I can now safely say that I prefer a statically typed language for complex algorithmic code.

During this time I kept reading about Rust and made a mental note that I should try it out sometime.
At some point I just went for it, wrote my first Hello World in Rust and started porting the Excel parsing logic of the Clojure program in Rust.
Now this is probably caused by lots of things other than just the pure performance characteristics of Clojure/the JVM vs Rust
, but when simply parsing the sheet took about 5 seconds in Clojure and 20ms in Rust, I was hooked.

I continued learning the basics of Rust and tried to port the existing algorithm as directly as possible.
Obviously, I was forced to redesign some parts, but in the beginning any redesign was made by necessity of the different paradigms between Clojure and Rust
, not a better understanding of the problem domain.
This stopped when I noticed a bug which actually made the old algorithm completely skip a few combinations of the input data, making its output a complete lie.
Finding this bug was definitely helped by having to rewrite and understand my old code, but in this case, it actually caused a compiler warning or error.
This was another situation where Rust won me over.

There are a few specific differences I want to showcase, namely error handling
, performance(FIXME I won't go deeper than anecdotal, so is the intro enough about performance?)
and readability/the type system.

Finally, I want to make some sweeping and completely subjective statements about those languages.

I've stopped regularly lurking around the Clojure community around March 2019, so my judgements might be not completely up to date
, looking at a recent (very good) article about clojure.spec (https://www.pixelated-noise.com/blog/2020/09/10/what-spec-is/#org6cf26d6)
, it seems like there were no big changes.

## Error Handling

My error handling experiences in this project were two-fold:
Firstly, I had a slightly complicated parsing and validation logic in which I made efforts to accumulate errors instead of aborting at the first one.
Secondly, the diff functionality was called by a web frontend, so I had to have some kind of main error handler in my request handling logic.

### Error Handling in Clojure

Error handling in Clojure is not opinionated.
Similar to error messages (https://lispcast.com/clojure-error-messages-accidental/)
, what error handling idioms exist in the Clojure community seem to me to be largely accidental/inherited from Java.
The standard library mostly supports exceptions (https://clojuredocs.org/clojure.core/ex-info).
There are some libraries that try to support error values similar to Rusts Result, like https://github.com/adambard/failjure.
Others, like `instaparse`, return their own custom result values, which is a totally sensible design, but it means that I can only handle `instaparse` errors by using `insta/failure?` (https://github.com/Engelberg/instaparse/blob/4d1903b059e77dc0049dfbc75af9dce995756148/README.md#parse-errors)

I used failjure to help accumulate errors in a nicer way (and because it appealed to my Haskell-influenced taste).
It worked, but I wasn't amazed by it. See for yourself in this example:

```clj
(defn parse
  [country-mapping expr]
  (fail/attempt-all
   [parse-result (insta/parse country-expr-parser expr)
    parse-ok (if (insta/failure? parse-result)
               (fail/fail
                (let [msg (str "Illegal country code expression " expr)]
                  {:msg msg
                   :log (str msg ":\n"
                             (with-out-str
                               (instaparse.failure/pprint-failure parse-result)))}))
               parse-result)
    countries (insta/transform
               {:expr apply-exclusions
                :country #(get country-mapping % #{%})}
               parse-ok)
    spec-ok (util/check-specs "Country"
                              identity
                              ::spec.common/country-code
                              countries)]
   spec-ok
   (fail/when-failed [failure]
                     (fail/fail
                      (let [msg (str "Failed to parse " expr)]
                        (merge-with #(str %1 "\n" %2)
                                    {:msg msg
                                     :log msg}
                                    (fail/message failure)))))))
```

Nice things about this:

* I can use fallible and infallible functions interchangeably (the call to `insta/parse` and to `insta/transform` will never return an error).
* I can optionally add an error handling function to the very end, which is helpful to, e.g., log the argument of the function, as I did here.

Not so nice things about this:

* I can't see which functions can actually fail, I have to read them.
* Since the Clojure ecosystem doesn't have a uniform error handling style, I have to manually convert exceptions or, in this case, the `instaparse` error type to `failjure` errors.

My verdict is:
Since Clojure is a Lisp, it's possible to use most kinds of error handling and make it look fine, if you really want to.
The most pragmatic solution in most cases will be to use exceptions
I found that readability can suffer when errors are are not handled explicitly, especially in a dynamic language.
I was tempted to experiment with different approaches, which gave me some experience but also took more time than if Clojure was more opinionated about the way errors should be handled.

### Error handling in Rust

Rust is quite opinionated about error handling.
The Rust community has worked on developing and improving common idioms, some of which were incorporated into the standard library, thereby advancing the baseline error handling.
There are lots of good articles about error handling in Rust (https://doc.rust-lang.org/book/ch09-00-error-handling.html, https://nick.groenen.me/posts/rust-error-handling/, https://blog.yoshuawuyts.com/error-handling-survey/, https://www.youtube.com/watch?v=rAF8mLI0naQ), which make learning the state of the art relatively easy (TODO wording, do I event want to make that point?)
Rust uses the `Result` type which contains errors as values, only using the exception-like `panic` for errors you probably don't want to recover from.
There are several libraries that make creating your own errors or handling errors from libraries easier, but they (mostly) just use the types from the standard library instead of introducing new stuff that's incompatible with the rest of the ecosystem.

I used `anyhow` to make aggregating errors returned from library functions seamless.

Code example below

TODO do I want to show the same spot as in Clojure?
Maybe I should find another function that shows off Rusts error handling better.

```rust
fn parse(country_mapping: &CountryMapping, input: &str) -> Result<Vec<CountryCode>> {
    let (_, (country_codes, exclusions)) = country_expr_parser(input)
        .map_err(|e| format_err!("Failed to parse {:?}: {}", input, e))?;

    let country_codes: Vec<_> = country_codes
        .into_iter()
        .flat_map(|code| {
            country_mapping
                .get(&code)
                .map(Clone::clone)
                .unwrap_or_else(|| vec![code])
        })
        .filter(|code| {
            if let Some(exclusions) = &exclusions {
                !exclusions.contains(&code)
            } else {
                true
            }
        })
        .map(CountryCode::new)
        .collect();
    Ok(country_codes)
}
```

## Performance

The type of performance problems I had were mostly being CPU bound, having to generate and compare a lot of data.
It also often caused memory issues, both in Clojure and in Rust.

### Clojure Performance

In Clojure, I often had to think about whether I should make my code less idiomatic to make it faster.
Many idioms and design approaches were a bad idea in hot loops (destructuring, creating a large amount of small maps by repeatedly calling assoc-in/update-in instead of creating POJOs - TODO link some articles to underline this point).
In the end I always had more option available - maybe even writing the hot part in Java - but many of these options would have made the code a lot less pretty.

TODO Code example?

TODO a verdict maybe? I think the initial text is enough

### Rust Performance

In Rust, designing for performance never felt unidiomatic (obviously, the language was designed to be very fast).
The hardest decisions were whether I should copy data around to make my life easier or use references and save memory but have to deal with lifetime annotations everywhere.
By the time I stopped developing the tool I still used unnecessary copies and wasn't really happy with the performance, so there's that ¯\_(ツ)_/¯

TODO code example? 

TODO a verdict?

## Readability

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

TODO ranty? not sure. muss editen

## Clojure vs Rust?

Even though I might be a bit of a Rust fanboy, I don't want to be unfair against Clojure here.

I definitely missed the Clojure REPL and Paredit after I stopped writing Clojure and I would love to have a similar experience in Kotlin or Rust.

The design approach of using a few elementary data structures for nearly everything and then manipulating those with functional programming can lead to wonderfully simple programs.

Nevertheless, my current preferences are: Rust for fun, keeping an eye open for opportunities where using Rust would be a clear improvement on the job, otherwise Kotlin.
