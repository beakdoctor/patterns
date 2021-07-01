---
title: "Chapter 1"
date: 2021-06-29T20:47:45+02:00
draft: true
---

This is an attempt to learn more about how people structure Clojure projects and
which patterns they commonly use. I started this with only basic knowledge
about Clojure and ClojureScript, but not without knowledge about the
Lisp-family of languages or functional programming in general. 

My goal here is to "mine" different projects for things to learn about Clojure. As such I may be a little all
over the place, searching for things to learn.

First out is the utility [kibit](https://github.com/jonase/kibit), which is a static analyzer for Clojure code
that detects code that can be made more idiomatic. How suitable!

Just to get an idea of what it's doing I ran the utility on another open-source project, asciinema-player.

```shell
$ lein kibit
At src/asciinema/player/core.cljs:26:
Consider using:
  (vt/feed-str (vt/make-vt (or width 80) (or height 24)) text)
instead of:
  (-> (vt/make-vt (or width 80) (or height 24)) (vt/feed-str text))

At dev/cljs/asciinema/player/dev.cljs:57:
Consider using:
  (.getElementById js/document "player")
instead of:
  (. js/document (getElementById "player"))

At dev/cljs/asciinema/player/dev.cljs:77:
Consider using:
  (.getElementById js/document "player")
instead of:
  (. js/document (getElementById "player"))
```

It doesn't approve of using the threading macro in this case, and suggests a different way of calling
the javascript function `getElementById`.


## First steps

After getting the project and the REPL up and running within Cursive I searched
for a suitable entry point to begin my studies, and landed on the file
`check.clj`, and the functions `check-file` and `check-expr`.

*The repository contains two sub-projects, and I ran into some crashes
with Cursive unless I opened the sub-projects separately such that
`project.clj` would be in the root of the IDEA project.*

## Pattern 1 - comment block

An alternative to directly inputting forms into the REPL is to create comment
forms. By putting them within the file itself we get to save illustrative
code blocks that we can easily go back to and evaluate whenever we want. I
choose to create this block to understand what these functions were doing.

```clojure
(comment
  (check-file "src/kibit/check.clj")
  (check-expr '(+ 1 a))
  (check-expr (list '+ 1 'a))
  (check-expr '(if true :a nil)))
```

Apparently they return a map with information about how the expression can be improved.

```clojure
(check-expr '(+ 1 a))
=> {:expr (+ 1 a)
    :line 213
    :column 16
    :end-line nil
    :end-column nil
    :alt (inc a)}
```

I was confused as to how it could know the line and column numbers of the expression, but since this is a Lisp I
shouldn't have been. Reading along I found the `meta` function, that returns the metadata of an object.

```clojure
(meta '(+ 1 a))
=> {:line 216, :column 10}
```

## Pattern 2 - associative destructuring

The function `check-file` had a let-expression that initially threw me off for a while. `kw-opts` is an argument passed
to the function.

```clojure
(let [{:keys [rules guard resolution reporter init-ns]
       :or   {reporter reporters/cli-reporter}}
      (merge default-args
             (apply hash-map kw-opts))]
  ...)
```

There are a few different ways of [destructuring](https://clojure.org/guides/destructuring) associative structures,
but in this case we extract the listed keys, and the `reporter` key gets a default value in case it is missing. The
other keys have default values from the `default-args` map. So in essence this is keyword args with default values.

## Pattern 3 - tree walking

`check-expr` applies simplifications deeply on the forms you pass to it.

```clojure
(check-expr '(if true (+ a 1) nil))
=> {:expr (if true (+ a 1) nil), ..., :alt (when true (inc a))}
```

It made me curious about the built-in facilities to traverse tree structures, and I found out about `walk` and
`tree-seq`.

Firstly, with `walk/prewalk` we can try to implement our own little simplifier.

```clojure
(let [simplify-fn #(if (= '(+ a 1) %) '(inc a) %)]
    (clojure.walk/prewalk simplify-fn '(if true (+ (+ a 1) (+ a 1)))))
=> (if true (+ (inc a) (inc a)))
```

It's very stupid, but it gets the job done. We can also get a sequence of the nodes in the tree by using `tree-seq`
if we want to extract some specific nodes. We could then build a simple reporter for our little simplifier:

```clojure
(let [simplify-fn #(if (= '(+ a 1) %) '(inc a) %)
      simplifiable #(not (= (simplify-fn %) %))]
  (map (fn [e] {:expr e :meta (meta e) :alt (simplify-fn e)})
       (filter simplifiable
               (tree-seq seq?
                         identity
                         '(if true (+ (+ a 1) (+ a 1)))))))
=>
({:expr (+ a 1), :meta {:line 228, :column 55}, :alt (inc a)}
 {:expr (+ a 1), :meta {:line 228, :column 63}, :alt (inc a)})
```

Wow, so simple!

## Pattern 4 - collections

Let's take a look at how the simplification rules are defined. Within the `rules` folder the rules are divided by the
type into different files, each containing a defrules form. Defrules is a macro defined within the project. Def* macros
are certainly very common in the Lisp world, but before looking at how they are defined, let's see what we can learn
from the predefined rules.

Here are some rules for collections:
```clojure
(defrules rules
  ;;vector
  [(conj [] . ?x) (vector . ?x)]
  [(into [] ?coll) (vec ?coll)]
  ...)
```

So, use the vector creation methods instead of `conj` or `into` into empty vectors.

Here is another rule:
```clojure
  ...
  [(not (empty? ?x)) (seq ?x)]
  ...
```

`seq` is apparently the idiomatic way of checking if a collection contains items. Not to be confused with `seq?` which
checks if its argument implements `ISeq`. Or `sequential?` which checks if the argument implements Sequential. What is
the difference? `ISeq` demands that the object implements `first`, `rest` and `cons`,
and `Sequential` indicates the object can be iterated over. For example, vectors are not `seq?` but they are
`sequential?`. Lists are both. Maps and sets are neither.

And what about `coll?`? It checks if the argument implements `IPersistentCollection`, which is true for all the above
mentioned data structures: vectors, lists, maps and sets.

## Pattern 5 - macros

Macros have become one of the defining features of Lisps. They work especially well because code is represented as
a data structure within the language itself. It will be interesting to see how they are utilized across
different Clojure projects. 

As mentioned the `defrules` construction is defined as a macro in kibit. Macros are handled a bit differently
across different Lisps, so let's check out how Clojure does them.

```clojure
(defmacro defrules [name & rules]
  `(let [rules# (for [rule# '~rules]
                  (if (raw-rule? rule#)
                    (eval rule#) ;; raw rule, no need to compile
                    (compile-rule rule#)))]
     (def ~name (vec rules#))))
```

You can almost guess what happens if you have seen Common Lisp macros. The backtick allows us to quote the form except
for the parts we want to be evaluated (for example: `~rules`). The symbol `#` gives us a shortcut to generate symbols
for our variables, so that we may avoid name clashes with surrounding code. You could also call `gensym` directly.

```clojure
(gensym "rules")
=> rules710911
```

The macro itself creates a new variable within the current namespace, with the specified name, containing a vector with
the specified rules. Each rule file (arithmetic.clj, collection.clj, etc.) has its own defrules declaration, and they
are collectively exposed through the file rules.clj.

I don't understand how the syntax of the rules is handled, or the use of `core.logic` in `compile-rule`, but it is
something I would like to revisit in a future post. This post is getting long enough.

## Conclusion

In this post I got to see a first example of a real-world Clojure project. I learned some practical details about
code exploration, destructuring of maps, tree walking, what defines different collection types, and macros.

I hope to continue with kibit in a future post, and understand why and how it's using the logic programming library.