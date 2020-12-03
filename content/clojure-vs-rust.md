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
, what error handling idioms exist in the Clojure community seem to me to be largely accidental.
The standard library mostly supports exceptions (https://clojuredocs.org/clojure.core/ex-info).
There are some libraries that try to support error values similar to Rusts Result, like https://github.com/adambard/failjure.
I didn't really see popular libraries use anything other than exceptions, which makes using failjure in an application a bit frustrating.

I used failjure to help accumulate errors in a nicer way, which worked, but I wasn't amazed by it.


Clojure:

```clj
(defn handle-diff
  [old-file new-file]
  (fail/attempt-all
    [[old-data new-data] (parse old-file new-file)
     result (diff/diff old-data new-data)]
    (-> (muuntaja/encode "application/transit+json" result)
        (response/response)
        (response/content-type "application/transit+json"))
    (fail/when-failed [failure]
                      (let [failure (fail/message failure)]
                        (if-let [log (:log failure)]
                          (log/warn (str "Diff failed\n" log)))
                        (response/bad-request (or (not-empty (:msg failure))
                                                  "Unknown failure - see logs!"))))))
```

Rust:

```rust
async fn diff_handler(
    diff_params: DiffOptions,
    form_data: multipart::FormData,
) -> Result<impl Reply, Error> {
    static OLD_SHEET_NAME: &str = "old";
    static NEW_SHEET_NAME: &str = "new";
    let mut forms = parse_form(form_data, &[OLD_SHEET_NAME, NEW_SHEET_NAME]).await?;
    let old_input = to_parse_input(forms.remove(OLD_SHEET_NAME).unwrap(), OLD_SHEET_NAME).await?;
    let new_input = to_parse_input(forms.remove(NEW_SHEET_NAME).unwrap(), NEW_SHEET_NAME).await?;

    match &diff_params.report {
        Some(s) if s == "drill-down" => run_diff::<DrillDown>(old_input, new_input, diff_params),
        _ => run_diff::<Summary>(old_input, new_input, diff_params),
    }
}
```

## Clojure vs Rust?

Even though I might be a bit of a Rust fanboy, I don't want to be unfair against Clojure here.
I definitely missed the Clojure REPL and Paredit in the beginning and would love to have a similar experience in Kotlin or Rust.
The design approach of using a few elementary data structures for nearly everything and then manipulating those with functional programming can lead to wonderfully simple programs.

Nevertheless, I'm firmly in the position where I use Rust for fun, I'm keeping an eye open for opportunities where using Rust would be a clear improvement, and otherwise Kotlin is my day job default.

