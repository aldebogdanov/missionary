# Hello flow

This tutorial will help you familiarize with the `flow` abstraction. A `flow` is a value representing a process able to produce an arbitrary number of values before terminating. Like `task`s, they're asynchronous under the hood and also support failure and graceful shutdown.


## Basic operations

Setup a dependency on the latest `missionary` release in your favorite environment and fire up a clojure REPL. Require `missionary`'s single namespace `missionary.core`, which will be aliased to `m` in the following.
```clojure
(require '[missionary.core :as m])
```

You can build a flow from an arbitrary collection with `seed`, and you can reduce `flow`s like collections with `reduce`, turning it into a `task`.
```clojure
;; A flow producing the 10 first integers
(def input (m/seed (range 10)))

;; A task producing the sum of the 10 first integers
(def sum (m/reduce + input))

(m/? sum)
#_=> 45
```

`eduction` passes a flow through a transducer.
```clojure
(m/? (m/reduce conj (m/eduction (partition-all 4) input)))
#_=> [[0 1 2 3] [4 5 6 7] [8 9]]
```

## Ambiguous evaluation

Not very interesting so far, because we haven't performed any action yet. Let's introduce the `ap` macro. `ap` is to `flow`s what `sp` is to `task`s. Like `sp`, it can be parked with `?`, but it has an additional superpower : it can be *forked*.
```clojure
(def hello-world
  (m/ap
    (println (m/?> (m/seed ["Hello" "World" "!"])))
    (m/? (m/sleep 1000))))

(m/? (m/reduce conj hello-world))
Hello
World
!
#_=> [nil nil nil]              
```

The `?>` operator pulls the first seeded value, forks evaluation and moves on until end of body, producing result `nil`, then *backtracks* evaluation to the fork point, pulls another value, forks evaluation again, and so on until enumeration is exhausted. Meanwhile, `reduce` consolidates each result into a vector. In an `ap` block, expressions have more than one possible value, that's why they're called *ambiguous process*.


## Preemptive forking

In the previous example, pulling a value from the flow passed to `?>` transfers evaluation control to the forked process, and waits for evaluation to be completed before pulling another value from the flow. In some cases though, we want the flow to keep priority over the forked process, so it can be shutdowned when more values become available. That kind of forking is implemented by `?<`.

We can use it to implement debounce operators. A debounced flow is a flow emitting only values that are not followed by another one within a given delay.
```clojure
(import 'missionary.Cancelled)
```

```clojure
(defn debounce [delay flow]
  (m/ap (let [x (m/?< flow)]                          ;; pull a value preemptively
    (try (m/? (m/sleep delay x))                      ;; emit this value after given delay
         (catch Cancelled _ (m/amb>))))))             ;; emit nothing if cancelled
```

To test it, we need a flow of values emitting at various intervals.
```clojure
(defn clock [intervals]
  (m/ap (let [i (m/?> (m/seed intervals))]
          (m/? (m/sleep i i)))))

(m/? (->> (clock [24 79 67 34 18 9 99 37])
          (debounce 50)
          (m/reduce conj)))
#_=> [24 79 9 37]
```

## Concurrent forking

What if we want to fork the processes concurrently? Use the `?>` operator with its extra `par` argument. It forks evaluation for `par` values concurrently, *all* values if you use the value `##Inf` for `par`. Values are returned from the flow in the order they finish, which is not necessarily the initial order.

```clojure
(m/? (m/reduce conj (m/ap (let [ms (m/?> ##Inf (m/seed [300 100 400 200]))]
                               (m/? (m/sleep ms ms))))))
#_=> [100 200 300 400]
```
