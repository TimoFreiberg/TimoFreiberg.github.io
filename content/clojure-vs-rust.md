+++
title = "I rewrote a Clojure program in Rust"
date = 2020-12-20
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

## Error Handling

The error handling requirements in this project were not very complicated.
All errors just needed to be logged and returned to the user.  
The only slightly unusual requirement was that parsing and validation logic should show all errors for each row in both uploaded excel files
(instead of just the first error) so I had to accumulate errors.

### Error Handling in Clojure

Error handling in Clojure is not opinionated.  
[Similar to error messages](https://lispcast.com/clojure-error-messages-accidental/)
, what error handling idioms exist in Clojure seem to me to be largely accidental or inherited from Java.

The standard library mostly supports [exceptions](https://clojuredocs.org/clojure.core/ex-info).  
There are some libraries that support returning error values instead of throwing exceptions like the error handling library [`failjure`](https://github.com/adambard/failjure).  
Others, like the parsing library [`instaparse`](https://github.com/Engelberg/instaparse), return their own custom error values[^insta-result].

I used failjure to help accumulate errors in a nicer way (and because it appealed to my Haskell-influenced taste).

Let's look at a Clojure function from my diff tool that uses [attempt-all](https://github.com/adambard/failjure#attempt-all) to parse and validate the input data.
If any errors occur, all errors are aggregated into a string:

```clj
(defn parse
  [country-mapping data]
  #_"   👇 the attempt-all function exits early 
           if any binding returned a failure"
  (fail/attempt-all
   [headers (header-row data)
    parsed (map
             #(parse-rule headers country-mapping %)
             (content-rows data))
    #_"           👇 list of failures is aggregated here"
    failed-parses (->> parsed
                    (filter fail/failed?)
                    (map fail/message))
    #_"          👇 this can return a failure,
                    triggering an early exit"
    parse-result (if (empty? failed-parses)
                   parsed
                   #_"👇 a single failure value containing the
                         list of failures concatenated into a string"
                   (fail/fail
                    (let [msg (str
                               "Failed to parse "
                               (count failed-parses)
                               " rules:")]
                      (str msg "\n" failed-parses))))
    #_"          👇 This can also return a failure"
    spec-result (util/check-specs "Rules"
                                  :rule/id
                                  ::spec/rule
                                  parse-result)]
   #_"👇 if everything was successful, this is returned"
   spec-result
   #_"👇 if any failure occurred, this is returned"
   (fail/when-failed [failure]
                       (do
                         (log/warn
                           (str "Failed to parse data "
                             data ":\n" (fail/message failure)))
                         failure))))
```

Nice things about this:

* The identifier-expression pairs in the square brackets use the same syntax as Clojure's [`let`-form](https://clojuredocs.org/clojure.core/let)
    which makes it look familiar.
* I can optionally add an error handling function to the very end, which is helpful to, e.g., log the argument of the function, as I did here.

Not so nice things about this:

* I can't see which functions can actually fail. I have to read them to find out.
* Since the Clojure ecosystem doesn't have a uniform error handling style, I have to manually convert exceptions or other errors like `instaparse` error values to `failjure` errors.

My verdict is:  
Since Clojure is a Lisp, it's possible to use most kinds of error handling and make it look fine, if you really want to.
The most pragmatic solution in most cases will be to use exceptions.

I found that readability could suffer when using less explicit error handling, especially in a dynamic language.

Due to the freedom of choice in error handling approacher, I was tempted to experiment more than with a more opinionated language.

### Error handling in Rust

Rust is quite opinionated about error handling.
The Rust community has worked on developing and improving common idioms, some of which were incorporated into the standard library, thereby improving the baseline error handling.

There's no improvement without change though, and the frequent changes have been a source of complaints.
While backwards compatibility was never broken, people that wanted their code to be idiomatic had to update it anyway.
Old tutorials and guides have therefore also become outdated.

There are lots of good, up-to-date articles about error handling in Rust[^rust-error-handling-links], which help learn the current idioms.

In Rust, functions that can error return the [`Result`](https://doc.rust-lang.org/std/result/index.html) type[^panic-ref].  
There are several libraries that make creating your own errors or handling errors from libraries easier, but they (mostly) just use the types from the standard library instead of introducing new stuff that's incompatible with the rest of the ecosystem.

Let's look at the same function as before, but this time in Rust:

```rust
pub fn parse(
    workbook: &mut Workbook,
    country_mapping: CountryMapping,
) -> Result<Vec<Rule>> {
    let range = workbook
        .worksheet_range("Rules")
        // this question mark triggers an early exit
        // there are two because we have an Option
        // containing a Result                    👇
        .ok_or(format_err!("Missing Rules sheet"))??;
        //                               👇
    let range = skip_to_header_row(range)?;
    let parsed = RangeDeserializerBuilder::new()
        .has_headers(true)
        .from_range(&range)
        //                                    👇
        .context("Failed to read Rules sheet")?;
    let rules = collect_errs(parsed.map(|parse_result| {
        parse_result
            // 👇 mapping a lambda over the error value
            .map_err(|e| e.into())
            // 👇 this would be called flatMap in some other languages
            .and_then(|row| row.parse(&country_mapping))
    }))
    // 👇 this converts a list of errors into a single error
    //    containing a string
    .map_err(|es| {
        format_err!(
            "Failed to parse {} rules:\n{}",
            es.len(),
            // 👇 very elegant...
            //    this would just be one .joinToString call in Kotlin
            es.into_iter()
                .map(|e| e.to_string())
                .collect::<Vec<_>>()
                .join("\n")
        )
   // 👇
    })?;
    Ok(rules)
}
```

Nice things about this:

* The standard library, every Rust library I've ever seen and my own application code is always using the same [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) type, which keeps things pretty compatible.
* [The `?` operator](https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html) makes fallible functions visible but keeps it succinct.  
  It also automatically converts error types where possible, which reduces the need for manual type conversion.
* The error type I'm using here from the library [`anyhow`](https://docs.rs/anyhow/*/anyhow/index.html) supports a [`.context`](https://docs.rs/anyhow/*/anyhow/trait.Context.html) method, which gives otherwise unhelpful low-level errors the necessary context.  
  This is usually done in exception-based languages by catching, wrapping and rethrowing, but this looks a lot more pleasant.

Not so nice things about this:

* I have to keep the error types compatible, which I accomplish in this case by not distinguishing between different error types at all[^anyhow-usecase].  
  I still have to return a single error value, which means I have to manually and verbosely convert the list of errors into a single one - in this case a newline-delimited string.
* If I want to log something when this entire function returns an error or add some `.context` to it, I would like to have the equivalent of a `try/catch`-block around the entire function body.  
  This doesn't exist yet[^try-blocks], the current best practice seems to be do move the entire body into an inner function or lambda.  
  
My verdict is:  
In Rust, you will use the `Result` type and you will like it[^and-you'll-like-it].  
The main design decisions are whether you use some of the helper libraries and how you design your error types.

Designing the error types can be a challenge though, especially because it's a bit different than designing e.g. Java exception hierarchies.  
I was lucky that keeping up to date with the evolving error handling idioms wasn't too hard for me as I was not under time pressure and often worked in my spare time with learning as my primary objective.
It might have been painful for teams maintaining bigger production systems.  
The large number of error handling tutorials and articles should hopefully make it easier to learn now than it was a few years ago.

The learning curve aside:
To me, Rust's error handling feels like part of the secret sauce that makes it the most promising language for correctness that I know of.

## Performance

The part of the program that caused performance issues was the diff algorithm and, to a slightly lesser extent, a data normalization step before that.  
The type of performance problems I had were mostly being CPU bound, having to generate and compare a lot of temporary data.
The large amount of data also often caused memory issues in both languages.

### Clojure Performance

In Clojure, making my code faster often meant making it less idiomatic.
Many of Clojure's idioms and design approaches are discouraged when optimizing code for performance[^clojure-goes-fast]:  
* using destructuring
* laziness
* the lazy sequence API
* using the usual Clojure hashmap as the main datastructure in hot loops.  

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
 #_"👇 a hashmap used in a (medium)-hot loop"
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
Unfortunately, doing that will make small details a bit uglier (when removing destructuring or the [`for`](https://clojuredocs.org/clojure.core/for) expressions).
Introducing records to replace all the little hashmaps with something faster would be a drop-in replacement, which is nice.

My verdict is:

Clojure successfully makes it possible to write fast code when necessary, but idiomatic Clojure willingly sacrifices some performance for expressiveness and its lispy dynamicity.  
This is fine.
Clojure does not want to be a low-level language.  
I was actually very positively surprised by all the escape hatches that were designed into the language to speed things up when necessary.  

I do like having the freedom to introduce abstractions that will have no runtime cost at all.
So let's look at Rust:

### Rust Performance

One of Rust's design goals was performance, and it [can](https://kornel.ski/rust-c-speed) [compete](https://github.com/ixy-languages/ixy-languages/blob/master/Rust-vs-C-performance.md) with C, so I guess that means it succeeded.  
But this doesn't automatically make my program the fastest - I can write slow code in any language!  
What was more interesting to me was how fast my program was going to be if I got it working and then spent one or two motivated weekends optimizing as best as I could.

Rust's [zero cost abstractions](https://boats.gitlab.io/blog/post/zero-cost-abstractions/) definitely helped with that.  
I could introduce wrapper types to make my type signatures more readable without affecting the performance at all.
I could use the functional [`Iterator` methods](https://doc.rust-lang.org/std/iter/trait.Iterator.html) like `map` and `filter` and get the same performance as if I was using imperative loops.

I could also use mutability freely under the watchful eye of Rusts type system, which can both improve performance and make things more readable.
Old-school imperative/OO languages like Java allowed mutability everywhere, which often caused problems[^mutability-problems].
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
                .intersection(&other.$field);

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
fn next_step<T: Eq + Ord + Hash, F: RuleField<'a, Item = T>>(
    &self,
    matcher: FieldMatcher<'a, T, F>,
) -> Self {
    // TODO according to heaptrack, this is a RAM hotspot. what do?
    let old_rules: Vec<_> = self
        .old_rules
        .iter()
        // 👇 this filter should be fast
        .filter(|vs| matcher.matches_inlined(vs))
        .copied()
        // 👇 heap allocation
        //    how about you do something about this, past Timo!
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

## Clojure vs Rust?

If it wasn't obvious before:
I have become quite a Rust fan and it's my preferred language to think in now.

I definitely missed the Clojure [REPL](https://clojure.org/guides/repl/introduction) and [Paredit](http://danmidwood.com/content/2014/11/21/animated-paredit.html) after I stopped writing Clojure and I would love to have a similar experience in Kotlin or Rust[^paredit-rust].  
The design approach of using a few elementary data structures for nearly everything and then manipulating those with functional programming can lead to wonderfully simple programs[^rust-not-simple].

Rust on the other hand is the most usable language that offers features that are otherwise only available in ML or Haskell.
Writing Rust can sometimes feel [as high level and productive as writing Kotlin](https://ferrous-systems.com/blog/rust-as-productive-as-kotlin/), but with a more expressive type system, a higher performance ceiling and fewer runtime restrictions[^runtime].

It's ownership and mutability tracking features also really help writing [correct software](https://fasterthanli.me/articles/aiming-for-correctness-with-types)
, and I miss those features in other languages.

## And what happened to that diff tool?

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

----

[^hooked-on-rust]: I was immediately hooked by the incredibly fast performance - which was initially mostly the difference between startup times and the respective excel parsing libraries, [docjure](https://github.com/mjul/docjure) and [calamine](https://github.com/tafia/calamine/).

[^not-hating-on-clojure]: I'm not trying to rip into Clojure here (after Rust and Kotlin it's my third favourite language).  
I'm trying to show how great Rust is by comparing it to a very good language.

[^insta-result]: This is a totally sensible design, but it means that I can only handle `instaparse` errors by using the `instaparse` [function `insta/failure?`](https://github.com/Engelberg/instaparse/blob/4d1903b059e77dc0049dfbc75af9dce995756148/README.md#parse-errors)

[^rust-error-handling-links]: See [the Rust Book](https://doc.rust-lang.org/book/ch09-00-error-handling.html), [this artice](https://blog.burntsushi.net/rust-error-handling/) by [ripgrep's](https://github.com/BurntSushi/ripgrep/) Andrew Gallant, [the error handling survey by Yoshua Wuyts](https://blog.yoshuawuyts.com/error-handling-survey/), [this article by Nick Groenen](https://nick.groenen.me/posts/rust-error-handling/) or [this talk by Jane Lusby](https://www.youtube.com/watch?v=rAF8mLI0naQ).

[^panic-ref]: There's also the [`panic` macro](https://doc.rust-lang.org/std/macro.panic.html) which works similar to exceptions in that it stops and unwinds your program, but unlike exceptions catching a panic is very rarely a good idea.

[^anyhow-usecase]: Handling all error types uniformly is `anyhow`s main usecase.

[^try-blocks]: I can't wait for [`try` blocks](https://doc.rust-lang.org/nightly/unstable-book/language-features/try-blocks.html) to be stabilized.

[^and-you'll-like-it]: Seriously, it's actually quite popular.

[^clojure-goes-fast]: See [these](https://tech.redplanetlabs.com/2020/09/02/clojure-faster/) [articles](http://clojure-goes-fast.com/blog/java-arrays-and-unchecked-math/) on how to improve Clojure performance.
They are, as far as I can tell, very accurate and contain good advice.
They also both discourage usual Clojure idioms or recommend less idiomatic alternatives.

[^mutability-problems]: This caused many Java programmers to avoid passing mutable collections around.

[^mutability-palatable]: Interestingly, some of my coworkers who are as fond of Kotlin as I am of Rust seem to have a much stronger aversion to mutability than I do.

[^borrow-redesign]: Some part of the code has to own the data to keep it alive while the hot loops use the references to it.

[^paredit-rust]: [Rust-analyzer](https://rust-analyzer.github.io/manual.html#extend-selection) and [IntelliJ](https://www.jetbrains.com/help/idea/working-with-source-code.html#editor_code_selection) support semantic extend/shrink selection though, which is an important feature of Paredit.
The slurp and barf features of Paredit probably don't make a lot of sense in languages without S-expressions anyway.

[^rust-not-simple]: Some parts, like the part where I was abstracting over fields of a datastructure in the diff algorithm, are noticeably more painful in Rust.

[^runtime]: Rust can be used in embedded systems, it can be compiled to WASM and it is very suitable for CLI tools where e.g. the JVM startup is painful.

