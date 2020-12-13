+++
title = "I rewrote a Clojure program in Rust"
date = 2020-12-13
+++

About two years ago, I wrote a quite complicated diff tool in Clojure.  
It was complicated enough that I struggled to fit the algorithm in my head and the inputs were large enough that I had to make some efforts to improve performance.

About half a year later, I started learning Rust, ported the current state of the Clojure program into Rust, was very happy with the change[^hooked-on-rust] and continued exclusively with Rust.  
While working on that project, I've developed some opinions about the two languages, especially about error handling and performance:

I think that these are areas where Rust excels, while they are among the weaker spots of Clojure[^not-hating-on-clojure].

To put my experience in context:
I had a bit more than one year of experience in Clojure when I moved to Rust.
The diff tool was by far the largest Clojure program I've ever written, and it was only about 3000 lines.  
When I started writing Rust, reimplementing the existing Clojure code was among my first Rust code.
I've continued learning Rust since then and have mostly stopped writing Clojure.  
If things have changed in Clojure recently, please let me know and I'll update the article.

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
, instead of just the first one, so I had to accumulate errors.

### Error Handling in Clojure

Error handling in Clojure is not opinionated.  
[Similar to error messages](https://lispcast.com/clojure-error-messages-accidental/)
, what error handling idioms exist in the Clojure community seem to me to be largely accidental/inherited from Java.

The standard library mostly supports [exceptions](https://clojuredocs.org/clojure.core/ex-info).  
There are some libraries that support returning error values instead of throwing exceptions like [the error handling library `failjure`](https://github.com/adambard/failjure).  
Others, like [the parsing library `instaparse`](https://github.com/Engelberg/instaparse), return their own custom result values[^insta-result].

I used failjure to help accumulate errors in a nicer way (and because it appealed to my Haskell-influenced taste).

Let's look at a Clojure function that parses and validates the input data, aggregating errors:

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
The Rust community has worked on developing and improving common idioms, some of which were incorporated into the standard library, thereby improving the baseline error handling.

There's no improvement without change though, and the frequent changes have been a source of complaints.
While backwards compatibility was never broken, people that wanted their code to be idiomatic had to update it anyway.
Old tutorials and guides have therefore also become outdated.

There are lots of good, up-to-date articles about error handling in Rust[^rust-error-handling-links], which help learn the current idioms.

In Rust, functions that can error return the [`Result` type](https://doc.rust-lang.org/std/result/index.html)[^panic-ref].  
There are several libraries that make creating your own errors or handling errors from libraries easier, but they (mostly) just use the types from the standard library instead of introducing new stuff that's incompatible with the rest of the ecosystem.

Let's look at the same function as before, but this time in Rust:

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
  I still have to return a single error value, which means I have to manually and verbosely convert the list of errors into a single one - in this case a newline-delimited string.
* If I want to log something when this entire function returns an error or add some `.context` to it, I would like to have the equivalent of a `try/catch`-block around the entire function body.  
  This doesn't exist yet[^try-blocks], the current best practice seems to be do move the entire body into an inner function or lambda.  
  
My verdict is:  
In Rust, you will use the `Result` type and you will like it[^and-you'll-like-it].  
The main design decisions are whether you use some of the helper libraries and how you design your error types.

Designing the error types can be a challenge though, especially because it's a bit different than designing e.g. Java exception hierarchies.  
I was lucky that keeping up to date with the evolving error handling idioms wasn't too hard for me as I was not under time pressure and often worked in my spare time to learn. It might have been painful for teams maintaining bigger production systems.  
The large number of error handling tutorials and articles should hopefully make it easier to learn now than it was a few years ago.

The learning curve aside:
To me, Rust's error handling feels like part of the secret sauce that makes it the most promising language for correctness that I know of.

## Performance

The part of the program that caused performance issues was the diff algorithm and, to a lesser extent, the data normalization step before that.  
The type of performance problems I had were mostly being CPU bound, having to generate and compare a lot of data.
The large amount of data also often caused memory issues in both languages.
Inefficiencies in the algorithm often caused both noticeable slowdowns and extreme memory issues at once.

### Clojure Performance

In Clojure, making my code faster often meant making it less idiomatic.
Many of Clojure's idioms and design approaches are discouraged when optimizing code for performance:  
using destructuring, laziness, the lazy sequence API, or using the usual Clojure hashmap as the main datastructure in hot loops[^clojure-goes-fast].  
In the end I always had more options available - maybe even writing the hot part in Java - but many of these options would have made the code a lot less pretty.

Let's look at one of the Clojure functions that were part of a medium-hot part of the program, where the input data was normalized:

```clojure
(defn rule-field-diff
  [rule1 rule2]
  (let [field-diff-fn (fn [operations]
                        (->>
                        #_"👇 laziness  👇 destructuring"
                          (for [{:keys [field op]} operations
                                :let [field1 (get rule1 field)
                                      field2 (get rule2 field)]]
                         #_"👇 a hashmap used in a (medium)-hot loop"
                            {:field field
                            :eq? (op field1 field2)
                            :left field1
                            :right field2})
                          #_"            👇 hashmap lookup"
                          (filter #(not (:eq? %)))
                          #_"    👇 hashmap operation"
                          (map #(dissoc % :eq?))))
        diff (field-diff-fn rule-must-be-equal-operations)
        mergeable-diff (field-diff-fn mergeable-rule-operations)]
    {:rule1 rule1
     :rule2 rule2
     :diff diff
     :mergeable-diff mergeable-diff}))
```

and here's one from the hottest part, where the data was diffed by one of its fields:

```clj
(defn diff-rules-by-keys
  #_"      👇 destructuring"
  [{:keys [group-by-key-fn key-name]} path rules1 rules2]
  #_"                👇 hashmap lookup"
  (let [key->rules1 (group-by-key-fn rules1)
        key->rules2 (group-by-key-fn rules2)
        keys1 (set (keys key->rules1))
        keys2 (set (keys key->rules2))
        key-union (set/union keys1 keys2)]
  #_"👇 laziness"
    (for [k key-union
          :let [path (conj path
                           {:key-name key-name
                            :key-val k})]]
      (case [(contains? keys1 k)
             (contains? keys2 k)]
        #_"          👇 hashmaps used in a hot loop"
        [true true] {::continue true
                     :path path
                     ::rules1 (get key->rules1 k)
                     ::rules2 (get key->rules2 k)}
        [false true] {:plus true
                      :path path
                      :rules (set (get key->rules2 k))}
        [true false] {:minus true
                      :path path
                      :rules (set (get key->rules1 k))}))))
```

So there are some obvious inefficiencies that might be worth it to change, great!
Unfortunately, doing that will make small details a bit uglier (when removing destructuring or the [for expressions](https://clojuredocs.org/clojure.core/for)) or require lots of additional changes (when replacing the hashmaps with records, for example).

My verdict is:

Clojure successfully makes it possible to write fast code when necessary, but idiomatic Clojure willingly sacrifices some performance for expressiveness and its lispy dynamicity.  
This is fine.
Clojure does not want to be a low-level language.  
I was actually very positively surprised by all the escape hatches that were designed into the language to speed things up when necessary.  

I do like having the freedom to introduce abstractions that will have no runtime cost at all.
So let's look at Rust:

### Rust Performance

One of Rust's design goals was performance, and it [can](https://kornel.ski/rust-c-speed) [compete](https://github.com/ixy-languages/ixy-languages/blob/master/Rust-vs-C-performance.md) with C.  
But this doesn't automatically make my program the fastest - I can write slow code in any language!  
What was more interesting to me was how fast my program was going to be if I got it working and then spent one or two motivated weekends optimizing as best as I could.

Rust's [zero cost abstractions](https://boats.gitlab.io/blog/post/zero-cost-abstractions/) definitely helped with that.  
I could introduce wrapper types to make my type signatures more readable without affecting the performance at all.
I could use the functional [`Iterator` methods](https://doc.rust-lang.org/std/iter/trait.Iterator.html) like `map` and `filter` and get the same performance as if I was using imperative loops.

I could also use mutability freely under the watchful eye of Rusts type system, which can both improve performance and make things more readable.
Oldschool imperative/OO languages like Java allowed mutability everywhere, which often caused problems[^mutability-problems].
Functional languages like Haskell and Clojure then tried to solve the problems with mutability by removing mutability as much as possible.  
Rust has surprised me by making mutability palatable again[^mutability-palatable].

The area where Rust programmers [often](https://nnethercote.github.io/perf-book/heap-allocations.html?highlight=borrow#cow) [have](https://deterministic.space/secret-life-of-cows.html#no-needless-copying) [to](https://llogiq.github.io/2017/06/01/perf-pitfalls.html) [decide](https://www.reddit.com/r/rust/comments/adyd9j/why_is_the_rust_version_of_this_fn_60_slower_than/edl0bly/?utm_source=reddit&utm_medium=web2x&context=3) between simple code and performance is when deciding between needlessly copying data or using references to the data.  
Using references to a single instance of data can be wildly more performant, but it requires lifetime annotations and sometimes requires redesigning the code[^borrow-redesign].

Let's look at one of the functions in the medium-hot data normalization part again:

```rust 
    //                  👇  mutability here! it's fine though
    pub fn intersection(mut self, other: &Self) -> Option<Self> {

        match (self.$field.accepts_all(), other.$field.accepts_all()) {
            // use the more restrictive field
            (true, false) => {
                *self.$field = other.$field.clone();
            }
            // self is more constrained, so we keep self
            (false, true) => {}
            // both accept everything, we can keep self
            (true, true) => {}
            // both are constrained, we need to calculate the intersection
            (false, false) => {
                let intersection = self.$field
                    .into_iter()
                    // 👇 functional iterator API
                    .filter(|x| other.$field.contains(x))
                    .collect::<Vec<_>>();

                // empty intersection -> unsatisfiable
                if intersection.is_empty() {
                    // 👇 sometimes, imperative logic is nice!
                    return None;
                }
                // 👇 mutability again
                self.$field = intersection;
            }
        }

        // if we didn't return None early, the field must be satisfiable!
        Some(self)
    }
```

Note the `$field` in the body - this was actually inside a macro definition!
Parameterizing the logic by field access and type was a bit clunky in Rust.
Similar functions in the Clojure version just take a keyword argument.

Let's now look at one of the hottest functions in the Rust implementation:

```rust
    fn next_step<T: Debug + PartialEq + Ord + Hash, F: RuleField<'a, Item = T>>(
        &self,
        matcher: FieldMatcher<'a, T, F>,
    ) -> Self {
        //TODO according to heaptrack, this is a RAM hotspot. what do?
        let old_rules: Vec<_> = self
            .old_rules
            .iter()
            // 👇 this filter should be fast
            .filter(|vs| matcher.matches_inlined(vs))
            .copied()
            // 👇 heap allocation, maybe do something about this, past Timo!
            .collect();
        let new_rules: Vec<_> = self
            .new_rules
            .iter()
            .filter(|vs| matcher.matches_inlined(vs))
            .copied()
            .collect();

        //                       👇 the path values are very small, but
        //                         I'm still cloning inside a hot loop
        let mut path = self.path.clone();
        matcher.add_step_inlined(&mut path);

        DiffState {
            path,
            old_rules,
            new_rules,
        }
    }
```

There are some inefficiencies visible here, and they're probably the most important spots for performance improvements.
But they're still there as fixing them was too hard and/or time-consuming for me.

My verdict is:

I think it's a testament to the language that I could write an algorithm I could barely understand and get the performance to a state where the performance problems were caused by the amount of data I was intentionally creating and processing (instead of incidental inefficiencies).


<!-- 
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

TODO ranty? not sure. need to edit -->

## Clojure vs Rust?

If it wasn't obvious before:
I have become quite a Rust fan and it's my preferred language to think in now.

I definitely missed the Clojure [REPL](https://clojure.org/guides/repl/introduction) and [Paredit](http://danmidwood.com/content/2014/11/21/animated-paredit.html) after I stopped writing Clojure and I would love to have a similar experience in Kotlin or Rust[^paredit-rust].  
The design approach of using a few elementary data structures for nearly everything and then manipulating those with functional programming can lead to wonderfully simple programs[^rust-not-simple].

Rust on the other hand is the most usable language that offers features that are otherwise only available in ML or Haskell.
Writing Rust can sometimes feel [as high level and productive as writing Kotlin](https://ferrous-systems.com/blog/rust-as-productive-as-kotlin/), but with a more expressive type system, a higher performance ceiling and fewer runtime restrictions[^runtime].

### And what happened to that diff tool?

After switching to Rust, I had to implement more complicated logic that resolved dependencies between the rules I was diffing.
This became complicated enough that even with a static type system I could only barely understand it.
I doubt I would have been able to finish it in Clojure.

In the end, I reached a stage where the tool seemed to work fine.
I managed to improve the performance to acceptable levels and then stopped working on the tool.
My colleagues who I wrote the tool for continued occasionally using it for over a year without any complaints.

The whole system whose input my tool was diffing is (hopefully) going to be sundowned soon and the replacement system will offer a simpler way to analyse changes:  
It will maintain a database counting the actual entities that exist, indexed by the fields that the rules apply to.

My diff tool will be replaced by a database lookup and I am very happy about that.

----

[^hooked-on-rust]: I was immediately hooked by the incredibly fast performance - which was initially mostly the difference between startup times and the respective excel parsing libraries.

[^not-hating-on-clojure]: I'm not going to rip into Clojure here (after Rust and Kotlin it's my third favourite language).  
I'm going to use a good language to show how great Rust is.

[^insta-result]: This is a totally sensible design, but it means that I can only handle `instaparse` errors by using the `instaparse` [function `insta/failure?`](https://github.com/Engelberg/instaparse/blob/4d1903b059e77dc0049dfbc75af9dce995756148/README.md#parse-errors)

[^rust-error-handling-links]: See [the Rust Book](https://doc.rust-lang.org/book/ch09-00-error-handling.html), [this artice](https://blog.burntsushi.net/rust-error-handling/) by [ripgrep's](https://github.com/BurntSushi/ripgrep/) Andrew Gallant, [the error handling survey by Yoshua Wuyts](https://blog.yoshuawuyts.com/error-handling-survey/), [this article by Nick Groenen](https://nick.groenen.me/posts/rust-error-handling/) or [this talk by Jane Lusby](https://www.youtube.com/watch?v=rAF8mLI0naQ).

[^panic-ref]: There's also the [`panic` macro](https://doc.rust-lang.org/std/macro.panic.html) which works similar to exceptions in that it stops and unwinds your program, but unlike exceptions catching a panic is very rarely a good idea.

[^anyhow-usecase]: Handling all error types uniformly is `anyhow`s main usecase.

[^try-blocks]: I can't wait for [`try` blocks](https://doc.rust-lang.org/nightly/unstable-book/language-features/try-blocks.html) to be stabilized.

[^and-you'll-like-it]: Seriously, it's quite popular.

[^clojure-goes-fast]: See [these](https://tech.redplanetlabs.com/2020/09/02/clojure-faster/) [articles](http://clojure-goes-fast.com/blog/java-arrays-and-unchecked-math/) on how to improve Clojure performance.
They are, as far as I can tell, very accurate and contain good advice.
They also both discourage usual Clojure idioms or recommend less idiomatic alternatives.

[^mutability-problems]: This caused many Java programmers to avoid passing mutable collections around.

[^mutability-palatable]: Interestingly, some of my coworkers who are as much Kotlin fans as I am a Rust fan seem to have a much stronger aversion to mutability than me.

[^borrow-redesign]: Some part of the code has to own the data to keep it alive while the hot loops use the references to it.

[^paredit-rust]: [Rust-analyzer](https://rust-analyzer.github.io/manual.html#extend-selection) and [IntelliJ](https://www.jetbrains.com/help/idea/working-with-source-code.html#editor_code_selection) support semantic extend/shrink selection though, which is an important feature of Paredit.
The slurp and barf features of Paredit probably don't make a lot of sense in languages without S-expressions anyway.

[^rust-not-simple]: Some parts, like the part where I was abstracting over fields of a datastructure in the diff algorithm, are noticeably more painful in Rust.

[^runtime]: Rust can be used in embedded systems, it can be compiled to WASM and it is very suitable for CLI tools where e.g. the JVM startup is painful.

