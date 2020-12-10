+++
title = "Rust excels where Clojure is lacking"
#Clickbaity enough?
date = 2020-12-02
+++

A while ago, I wrote a quite complicated diff tool in Clojure.
It was complicated enough that I struggled to fit the algorithm in my head and the inputs were large enough that I had to make some efforts to improve performance.
After a while, I started learning Rust, ported the current state of the Clojure program into Rust, was very happy with the change[^hooked-on-rust] and continued exclusively with Rust.
While working on that project, I've developed some opinions about the two languages, especially about error handling, performance and readability.

I think that these are areas where Rust excels, while they are among the weaker spots of Clojure[^not-hating-on-clojure-tho].

To put my experience in context:
I had a bit more than one year of experience in Clojure when I moved to Rust.
The diff tool was by far the largest Clojure program I've ever written, and it was only about 3000 lines.  
When I started writing Rust, reimplementing the existing Clojure code was among my first Rust code.
I've continued learning Rust since then and have mostly stopped writing Clojure.
If things have changed in Clojure recently, please let me know and I'll update the article.

TODO some more intro before starting with error handling? Or a nice segue?

<!-- 
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
, it seems like there were no big changes. -->

## Error Handling

The error handling requirements in this project were not very complicated.
All errors just need to be logged and returned to the user.
The only slightly unusual requirement was that parsing and validation logic should show all errors for each row in both uploaded excel files
, instead of just the first one.

### Error Handling in Clojure

Error handling in Clojure is not opinionated.
[Similar to error messages](https://lispcast.com/clojure-error-messages-accidental/)
, what error handling idioms exist in the Clojure community seem to me to be largely accidental/inherited from Java.

The standard library mostly supports [exceptions](https://clojuredocs.org/clojure.core/ex-info).  
There are some libraries that support returning error values instead of throwing exceptions like [the error handling library `failjure`](https://github.com/adambard/failjure).  
Others, like [the parsing library `instaparse`](https://github.com/Engelberg/instaparse), return their own custom result values[^insta-result].

I used failjure to help accumulate errors in a nicer way (and because it appealed to my Haskell-influenced taste).
It worked, but I wasn't amazed by it. See for yourself in this example:

```clj
(defn parse
  [country-mapping data]
  (fail/attempt-all
   [headers (header-row data)
    parsed (map #(parse-rule headers country-mapping %)
                (content-rows data))
    failed-parses (->> parsed
                       (filter fail/failed?)
                       (map fail/message))
    parse-result (if (empty? failed-parses)
                   parsed
                   (fail/fail
                    (let [msg (str
                               "Failed to parse "
                               (count failed-parses)
                               " rules:")]
                      (apply merge-with #(str %1 "\n" %2)
                             {:msg msg
                              :log msg}
                             failed-parses))))
    spec-result (util/check-specs "Rules"
                                  :rule/id
                                  ::spec/rule
                                  parse-result)]
   spec-result
   (fail/when-failed [failure]
                       (do
                         (if-let [log (:log (fail/message failure))]
                           (log/warn (str "Failed to parse data " data ":\n" log)))
                         failure))))
```

Nice things about this:

* I can use fallible and infallible functions interchangeably (the call to `map` will never return an error).
* I can optionally add an error handling function to the very end, which is helpful to, e.g., log the argument of the function, as I did here.

Not so nice things about this:

* I can't see which functions can actually fail, I have to read them.
* Since the Clojure ecosystem doesn't have a uniform error handling style, I have to manually convert exceptions or other errors like `instaparse` error values to `failjure` errors.

My verdict is:  
Since Clojure is a Lisp, it's possible to use most kinds of error handling and make it look fine, if you really want to.
The most pragmatic solution in most cases will be to use exceptions.
I found that readability can suffer when errors are not handled explicitly, especially in a dynamic language.
I was tempted to experiment with different approaches, which gave me some experience but also took more time than if Clojure was more opinionated about the way errors should be handled.

### Error handling in Rust

Rust is quite opinionated about error handling.
The Rust community has worked on developing and improving common idioms, some of which were incorporated into the standard library, thereby advancing the baseline error handling.
This instability has been painful for some people, TODO
There are lots of good articles about error handling in Rust[^rust-error-handling-links], which make learning the state of the art relatively easy. (TODO wording, do I even want to make that point?)

Rust uses the `Result` type which contains errors as values, only using the exception-like `panic` for errors you probably don't want to recover from.
There are several libraries that make creating your own errors or handling errors from libraries easier, but they (mostly) just use the types from the standard library instead of introducing new stuff that's incompatible with the rest of the ecosystem.

I used `anyhow` to make aggregating errors returned from library functions seamless.


```rust
pub fn parse(
    workbook: &mut Workbook,
    country_mapping: CountryMapping,
) -> Result<Vec<Rule>> {
    let range = workbook
        .worksheet_range("Rules")
        .ok_or(format_err!("Missing Rules sheet"))??;
    let range = skip_to_header_row(range)?;
    let parsed = RangeDeserializerBuilder::new()
        .has_headers(true)
        .from_range(&range)
        .context("Failed to read Rules sheet")?;
    let rules = collect_errs(parsed.map(|parse_result| {
        parse_result
            .map_err(|e| e.into())
            .and_then(|row| row.parse(&country_mapping))
    }))
    .map_err(|es| {
        format_err!(
            "Failed to parse {} rules:\n{}",
            es.len(),
            es.into_iter()
                .map(|e| e.to_string())
                .collect::<Vec<_>>()
                .join("\n")
        )
    })?;
    Ok(rules)
}
```

Nice things about this:

* The standard library, every Rust library I've ever seen and my own application code is always using the same [`Result` type](https://doc.rust-lang.org/std/result/enum.Result.html), which keeps things pretty compatible.
* [The `?` operator](https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html) makes the error handling visible but succinct.  
  It also automatically converts error types where possible, which reduces the need for manual type conversion.
* The error type I'm using here from the [library `anyhow`](https://docs.rs/anyhow/*/anyhow/index.html) supports [a `.context` method](https://docs.rs/anyhow/*/anyhow/trait.Context.html), which makes otherwise unhelpful low-level errors usable.  
  This is usually done in Exception-based languages by catching, wrapping and rethrowing exceptions, but this looks a lot more pleasant.

Not so nice things about this:

* I have to keep the error types compatible, which I accomplish in this case by not distinguishing between different error types at all[^anyhow-usecase].  
  I still have to return a single error value, which means I have to manually and verbosely convert the list of errors into a single, newline-delimited string.
* If I want to log something when this entire function returns an error or add some `.context` to it, the current best practice seems to be do move the entire body into an inner function `inner` and then call `inner(...).map_err(|e| ...)`.  
  I can't wait for [`try` blocks](https://doc.rust-lang.org/nightly/unstable-book/language-features/try-blocks.html) to be stabilized.

My verdict is:  
In Rust, you will use the `Result` type and you will like it[^and-you'll-like-it].  
The main design decisions are whether you use some of the helper libraries and how you design your error types.
There's enough writing about this topic that I quickly felt comfortable about my decisions because I knew that they were idiomatic (or at least close enough).


## Performance

The type of performance problems I had were mostly being CPU bound, having to generate and compare a lot of data.
The large amount of data also often caused memory issues in both languages.
Inefficiencies in the algorithm often caused both noticeable slowdowns and extreme memory issues at once.

### Clojure Performance

In Clojure, I often had to think about whether I should make my code less idiomatic to make it faster.
Many idioms and design approaches are discouraged in hot loops (destructuring, creating a large amount of small maps by repeatedly calling assoc-in/update-in instead of creating POJOs - TODO link some articles to underline this point).
In the end I always had more options available - maybe even writing the hot part in Java - but many of these options would have made the code a lot less pretty.

TODO I don't think I will show code examples here

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

TODO ranty? not sure. need to edit

## Clojure vs Rust?

Even though I might be a bit of a Rust fanboy, I don't want to be unfair against Clojure here.

I definitely missed the Clojure REPL and Paredit after I stopped writing Clojure and I would love to have a similar experience in Kotlin or Rust.

The design approach of using a few elementary data structures for nearly everything and then manipulating those with functional programming can lead to wonderfully simple programs.

Rust on the other hand is the most usable language that offers features that are otherwise only available in ML or Haskell.
Writing Rust can sometimes feel as highlevel and productive as writing Kotlin[^rust-vs-kotlin], but with a more expressive type system and the option to go as low-level is I want.

Nevertheless, my current preferences are: Rust for fun, keeping an eye open for opportunities where using Rust would be a clear improvement on the job, otherwise Kotlin.

----

[^hooked-on-rust]: I was immediately hooked by the incredibly fast performance - which was initially mostly the difference between startup times and the respective excel parsing libraries.

[^not-hating-on-clojure-tho]: I don't want to rip into Clojure here, after Rust and Kotlin it's my third favourite language.

[^insta-result]: This is a totally sensible design, but it means that I can only handle `instaparse` errors by using the `instaparse` [function `insta/failure?`](https://github.com/Engelberg/instaparse/blob/4d1903b059e77dc0049dfbc75af9dce995756148/README.md#parse-errors)

[^rust-error-handling-links]: See [the Rust Book](https://doc.rust-lang.org/book/ch09-00-error-handling.html), [this blog post](https://blog.burntsushi.net/rust-error-handling/) by [ripgrep's](https://github.com/BurntSushi/ripgrep/) Andrew Gallant, [the error handling survey by Yoshua Wuyts](https://blog.yoshuawuyts.com/error-handling-survey/), [this blog post by Nick Groenen](https://nick.groenen.me/posts/rust-error-handling/) or [this talk by Jane Lusby](https://www.youtube.com/watch?v=rAF8mLI0naQ).

[^anyhow-usecase]: Handling all error types uniformly is `anyhow`s main usecase.

[^and-you'll-like-it]: Seriously, it's quite popular.

[^rust-vs-kotlin]: See [this article](https://ferrous-systems.com/blog/rust-as-productive-as-kotlin/) by Aleksey Kladov for a more thorough comparison between Kotlin and Rust.

