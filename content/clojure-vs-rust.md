+++
title = "Clojure vs Rust"
date = 2020-12-02
+++


Points I want to touch on:

* Error handling
* Performance
* Readability/Type System

## Intro

I was quite into Clojure a while ago and was looking for something to practice on.
So I started writing something about some relatively complicated masterdata that some of my colleagues recently started to work with.

I started writing some parsing and validation logic and exploring the domain.
Inspired by some prod issues caused by accidental maintenance errors in this data
(which was maintained in Excel sheets [link to the excel fails during the corona crisis])
, I proposed writing a semantic differ to compare two versions of the excel sheet.

And so began the most algorithmically complicated program I have written so far.


## Error Handling

Clojure:

```clj
(defn handle-diff
  [request]
  (let [{:keys [old-file new-file]} (get request :params)]
    (verify-file-size old-file)
    (verify-file-size new-file)
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
                                                   "Unknown failure - see logs!")))))))
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
