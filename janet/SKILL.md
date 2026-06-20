---
name: janet
description: Write idiomatic Janet code. Use when writing, refactoring, or reviewing Janet (.janet) code. Covers functional patterns, performance tradeoffs, common gotchas, data structure idioms, PEG parsing, and Janet-specific pitfalls.
---

# Janet Style Guide

This skill guides the generation of idiomatic Janet code covering functional patterns, performance tradeoffs, common gotchas, and Janet-specific idioms.

## Core Principles

### Functional Style by Default
- **Always prefer functional style** unless there is a clear performance penalty or tradeoff
- Use imperative/mutable approaches only when performance demands it (large collections, hot paths, string building)
- When mutation is used for performance, document why and keep scope limited

### Prefer `let` Over Sequential `def`
Use `let` to group related bindings instead of sequential `def` statements:
```janet
# Good - let groups bindings clearly
(defn process [data]
  (let [items (data :items)
        count (length items)
        filtered (filter valid? items)]
    (map transform filtered)))

# Avoid - sequential defs
(defn process [data]
  (def items (data :items))
  (def count (length items))
  (def filtered (filter valid? items))
  (map transform filtered))
```
Reserve `def` for top-level definitions and cases where bindings are interspersed with side effects or control flow that makes `let` awkward.

### Prefer Immutability (When Practical)
- Use `def` for immutable bindings instead of `var`
- Favor pure functions without side effects
- When mutation is necessary, limit scope and make it explicit

### Embrace Higher-Order Functions
Use Janet's rich set of functional primitives:
- `map`, `filter`, `reduce` for collection transformations
- `keep`, `keep-indexed` for filtering with transformation
- `mapcat` for flat-mapping operations
- `partition`, `take`, `drop` for sequence manipulation
- `comp`, `partial`, `complement` for function composition
- `juxt` for parallel application

### Favor Expressions Over Statements
- Use `if`, `cond`, `case` as expressions that return values
- Prefer `when` for single-branch conditionals with side effects
- Use `->>` and `->` threading macros for pipeline clarity
- Chain operations rather than using intermediate variables

### Use Destructuring
Destructure function parameters and let bindings for clarity:
```janet
# Good - destructuring
(defn process [{:name name :age age}]
  (string name " is " age))

# Avoid - manual extraction
(defn process [person]
  (def name (person :name))
  (def age (person :age))
  (string name " is " age))
```

## Common Functional Patterns

### Collection Transformation Pipelines
```janet
# Good - functional pipeline
(->> data
     (filter some-pred?)
     (map transform)
     (reduce combine init))

# Avoid - procedural loops
(var result init)
(each item data
  (when (some-pred? item)
    (set result (combine result (transform item)))))
```

### Building Data Structures
```janet
# Good - expression-based construction
(def users
  (map (fn [{:name n :age a}]
         {:name n :adult? (>= a 18)})
       raw-data))

# Avoid - imperative building
(var users @[])
(each person raw-data
  (array/push users
    {:name (person :name)
     :adult? (>= (person :age) 18)}))
```

### Conditional Logic
```janet
# Good - cond expression
(def status
  (cond
    (< score 60) :fail
    (< score 80) :pass
    :excellent))

# Avoid - nested ifs with mutation
(var status nil)
(if (< score 60)
  (set status :fail)
  (if (< score 80)
    (set status :pass)
    (set status :excellent)))
```

### Function Composition
```janet
# Good - compose smaller functions
(def process
  (comp
    (partial filter valid?)
    (partial map normalize)
    (partial sort compare)))

# Use threading for readability
(defn analyze [data]
  (->> data
       (filter valid?)
       (map normalize)
       (sort compare)
       (take 10)))
```

### Recursion and Reduce
```janet
# Good - tail-recursive
(defn factorial [n]
  (defn fac-iter [n acc]
    (if (<= n 1)
      acc
      (fac-iter (- n 1) (* n acc))))
  (fac-iter n 1))

# Good - reduce for aggregation
(defn sum [xs] (reduce + 0 xs))

# Avoid - imperative loop for simple aggregation
(defn sum [xs]
  (var total 0)
  (each x xs (set total (+ total x)))
  total)
```

## Janet-Specific Idioms

### Sequence Processing
```janet
# Use keep for filter+map
(keep |(when (even? $) (* $ 2)) (range 10))

# Use partition for chunking
(partition 3 (range 10))

# Use interleave/interpose for combining
(interleave [:a :b :c] [1 2 3])
```

### Short Functions
Use `|` short-fn syntax for concise anonymous functions:
```janet
(map |(* $ $) (range 5))        # square
(filter |(> $ 10) numbers)      # greater than 10
(reduce |(+ $0 $1) 0 numbers)   # sum
```

### Pattern Matching
Use `match` for elegant conditional dispatch:
```janet
(defn describe [value]
  (match value
    [:ok x] (string "success: " x)
    [:err e] (string "error: " e)
    _ "unknown"))
```

### Struct/Table Construction
```janet
# Good - literal construction
(def config
  {:host "localhost"
   :port 8080
   :debug true})

# Use struct for immutable maps
(struct :a 1 :b 2 :c 3)

# Use zipcoll for key-value pairing
(zipcoll [:a :b :c] [1 2 3])
```

## Performance: When to Choose Imperative Style

Janet's mutable data structures are often faster than immutable ones. Use imperative/mutable approaches when:

### High-Performance Scenarios
- **Building large collections incrementally**: Use `@[]` and `array/push` instead of repeated concatenation
- **Accumulating results in loops**: Local mutation with `var` is faster than recursive accumulation
- **Hot paths and tight loops**: Mutation avoids allocation overhead
- **String building**: Use `buffer` with `buffer/push-string` instead of string concatenation
- **Large data processing**: Mutable operations avoid copying large structures

### Performance-Optimized Patterns

**Fast array building:**
```janet
# Fast - mutation for large results
(defn process-large [items]
  (def result @[])
  (each item items
    (when (expensive-pred? item)
      (array/push result (expensive-transform item))))
  result)

# Slower for large collections - creates intermediate arrays
(defn process-large [items]
  (->> items
       (filter expensive-pred?)
       (map expensive-transform)))
```

**Fast string building:**
```janet
# Fast - buffer mutation
(defn build-report [data]
  (def buf @"")
  (each item data
    (buffer/push-string buf "Item: ")
    (buffer/push-string buf (item :name))
    (buffer/push-string buf "\n"))
  (string buf))

# Slower - string concatenation
(defn build-report [data]
  (reduce (fn [acc item]
            (string acc "Item: " (item :name) "\n"))
          "" data))
```

**Fast accumulation:**
```janet
# Fast - local mutation
(defn sum-squares [numbers]
  (var total 0)
  (each n numbers
    (+= total (* n n)))
  total)

# Slower - functional reduce (minor difference, but matters in hot paths)
(defn sum-squares [numbers]
  (reduce (fn [acc n] (+ acc (* n n))) 0 numbers))
```

**Fast table/struct construction:**
```janet
# Fast - build mutable then freeze
(defn group-by [key-fn items]
  (def groups @{})
  (each item items
    (def k (key-fn item))
    (if-let [group (groups k)]
      (array/push group item)
      (put groups k @[item])))
  (table/to-struct groups))  # Return immutable if needed
```

### When Functional Style Still Wins
- **Small collections**: Overhead is negligible, clarity matters more
- **One-pass transformations**: `map`/`filter` are well-optimized
- **Code that's not performance-critical**: Readability and maintainability trump speed
- **When immutability prevents bugs**: Thread safety, easier reasoning about code

### Hybrid Approach
Combine both styles for optimal results:
```janet
# Functional pipeline with imperative inner loop for performance
(defn analyze-data [raw-data]
  (->> raw-data
       (filter valid?)
       (partition-by get-category)
       (map (fn [batch]
              # Imperative processing of each batch
              (def result @{})
              (each item batch
                (def key (item :id))
                (put result key (expensive-compute item)))
              result))))
```

## Anti-Patterns to Avoid

### Dogmatic Functional Style Over Performance
```janet
# Avoid - pure but slow for large data
(defn process-huge-file [lines]
  (reduce (fn [acc line]
            (if (matches? line)
              (array/concat acc [(parse line)])
              acc))
          [] lines))

# Prefer - imperative but fast
(defn process-huge-file [lines]
  (def result @[])
  (each line lines
    (when (matches? line)
      (array/push result (parse line))))
  result)
```

### Over-using var and set for Simple Values
```janet
# Avoid - unnecessary mutation
(var x 0)
(set x 10)
(set x (+ x 5))

# Prefer - direct calculation
(def x (+ 10 5))
```

### Side Effects in Map/Filter (Unless Intentional)
```janet
# Avoid - side effects in map (unless debugging)
(map (fn [x] (print x) (* x 2)) items)

# Prefer - separate concerns
(each x items (print x))
(map |(* $ 2) items)

# OK for debugging/logging
(map |(do (log/debug "processing" $) (process $)) items)
```

### Deeply Nested Conditionals
```janet
# Avoid
(if a
  (if b
    (if c x y)
    z)
  w)

# Prefer cond
(cond
  (and a b c) x
  (and a b) y
  a z
  w)
```

## Janet Gotchas and Common Pitfalls

### Truthiness: nil and false
Only `nil` and `false` are falsey in Janet. Everything else is truthy, including:
```janet
(if 0 "truthy" "falsey")      # => "truthy" (0 is truthy!)
(if "" "truthy" "falsey")     # => "truthy" (empty string is truthy!)
(if [] "truthy" "falsey")     # => "truthy" (empty array is truthy!)
(if {} "truthy" "falsey")     # => "truthy" (empty table is truthy!)

# Use explicit checks for emptiness
(if (empty? arr) "empty" "not empty")
(if (zero? n) "zero" "not zero")
```

### Equality: = vs deep=
`=` checks reference equality for data structures, not content:
```janet
(= [1 2 3] [1 2 3])           # => false (different objects)
(deep= [1 2 3] [1 2 3])       # => true (same content)

(def a [1 2 3])
(def b a)
(= a b)                       # => true (same reference)

# Watch out in data structure comparisons
(def m {:a [1 2]})
(= (m :a) [1 2])              # => false! Use deep=
```

### Keywords vs Symbols
Keywords and symbols look similar but behave differently:
```janet
:keyword    # keyword - evaluates to itself
'symbol     # symbol - quoted, used for metaprogramming

# Keywords are functions that look themselves up
(:name {:name "Alice"})       # => "Alice"
(map :name users)             # common idiom

# Don't confuse them
(= :foo 'foo)                 # => false
```

### Nil Punning in Collections
`nil` values in tables/structs disappear:
```janet
(def m {:a 1 :b nil :c 3})
m                             # => {:a 1 :c 3} (no :b!)
(get m :b)                    # => nil
(in m :b)                     # => false (key doesn't exist)

# Use sentinel values if you need to distinguish
(def m {:a 1 :b :missing :c 3})
```

### Function Arity: Variable Arguments
Functions with variable arguments have gotchas:
```janet
# & rest parameters capture remaining args
(defn f [a b & rest]
  (pp rest))

(f 1 2 3 4)                   # rest => @[3 4] (mutable array!)

# Named parameters must come before &
(defn bad [& rest a b]        # WRONG - syntax error
  ...)

(defn good [a b & rest]       # Correct order
  ...)
```

### Array/Tuple Confusion
Arrays and tuples have different properties:
```janet
[1 2 3]      # tuple (immutable)
@[1 2 3]     # array (mutable)
(tuple 1 2 3) # tuple (immutable, same as [1 2 3])

# Bracket literals are tuples, NOT arrays
(type [1 2 3])                # => :tuple
(type @[1 2 3])               # => :array

# You can't mutate a tuple
(array/push [1 2 3] 4)        # ERROR - not an array

# Use @[] when you need mutability
(def a @[1 2 3])
(array/push a 4)              # OK
```

### Struct vs Table: Immutability Surprise
```janet
(def s {:a 1})                # struct (immutable)
(put s :b 2)                  # ERROR: struct is immutable

(def t @{:a 1})               # table (mutable)
(put t :b 2)                  # OK

# Converting between them
(table/to-struct t)           # table -> struct (freezes)
(struct/to-table s)           # struct -> table (thaws)
```

### Integer Division vs Float Division
```janet
(/ 5 2)                       # => 2.5 (float division)
(div 5 2)                     # => 2 (integer division)

# Watch out in loops
(for i 0 (/ 10 3)             # i goes 0, 1, 2 (up to 3.333...)
  (print i))

(for i 0 (div 10 3)           # i goes 0, 1, 2 (up to 3)
  (print i))
```

### Scope and def vs var
```janet
# def creates new binding in current scope
(def x 10)
(let [x 20]
  (def x 30)                  # New binding in let scope
  x)                          # => 30
x                             # => 10 (outer unchanged)

# var allows mutation
(var x 10)
(let []
  (set x 30)                  # Mutates outer var
  x)                          # => 30
x                             # => 30 (outer changed)
```

### Short Function $ Parameters
```janet
# $ is first arg, $0 is also first arg
|(+ $ 1)        # (fn [x] (+ x 1))
|(+ $0 1)       # same thing

# $1, $2, etc. for multiple args
|(+ $0 $1)      # (fn [x y] (+ x y))

# Can't mix $ and explicit parameters
|(fn [x] (+ $ x))  # ERROR
```

### do Block Returns Last Value
```janet
(def x (do
         (print "calculating")
         (+ 1 2)
         (* 3 4)))            # x => 12 (last value)

# Watch out with conditionals
(if (even? n)
  (do
    (print "even")
    :even)                    # Return :even
  :odd)

# Missing return value
(if (even? n)
  (print "even")              # Returns nil!
  :odd)
```

### Loop vs While vs Each
```janet
# loop is a general iteration macro with verbs
(loop [i :range [0 10]]
  (print i))

# loop with multiple bindings and conditionals
(loop [i :range [0 10]
       :when (even? i)]
  (print i))

# loop with :in for iterating collections
(loop [x :in items
       :when (pos? x)]
  (print x))

# while is for conditional loops
(var i 0)
(while (< i 10)
  (+= i 1))

# each is shorthand for (loop [x :in coll] ...)
(each x items
  (print x))

# Prefer each for simple iteration, loop when you need
# :range, :keys, :pairs, :when, :until, :let, etc.
```

### Negative Indices Work Differently
```janet
(def a [1 2 3 4 5])
(a -1)                        # => 5 (last element)
(a -2)                        # => 4 (second to last)

# But get doesn't support negative indices
(get a -1)                    # => nil (not 5!)

# Use negative indices carefully
```

### Macro Hygiene and Unquote
```janet
# Macros don't evaluate arguments
(defmacro bad [x]
  (def y 10)
  ~(+ ,x y))                  # y might capture user's y

# Use gensym for hygiene
(defmacro good [x]
  (def y-sym (gensym))
  ~(let [,y-sym 10]
     (+ ,x ,y-sym)))

# Usually use defn, not defmacro
```

### Array Mutation Surprise
```janet
(def original @[1 2 3])
(def copy original)           # Not a copy! Same reference

(array/push copy 4)
original                      # => @[1 2 3 4] (mutated!)

# Actually copy
(def real-copy (array/slice original))
(array/push real-copy 5)
original                      # => @[1 2 3 4] (unchanged)
```

### reduce Argument Order
```janet
# reduce is: (reduce func init collection)
(reduce + 0 [1 2 3])          # correct

# Easy to get wrong
(reduce [1 2 3] + 0)          # WRONG - errors

# Function receives (accumulator, value)
(reduce (fn [acc x] (+ acc x)) 0 [1 2 3])
```

### PEG (Parsing Expression Grammar) Capture Gotchas
```janet
# PEG captures can be subtle
(peg/match '(capture (some :d)) "123")   # => @["123"]
(peg/match '(some (capture :d)) "123")   # => @["1" "2" "3"]

# Position vs capture
(peg/match '(* "a" (position)) "abc")    # => @[1]
(peg/match '(* "a" (capture "b")) "abc") # => @["b"]
```

## Code Review Checklist

When generating or reviewing Janet code, verify:
- [ ] `let` used for grouped bindings instead of sequential `def`s
- [ ] `[...]` used for tuples (immutable), `@[...]` for arrays (mutable)
- [ ] `{...}` used for structs (immutable), `@{...}` for tables (mutable)
- [ ] String building uses `buffer` for multiple concatenations, not repeated `string`
- [ ] Large collection building uses `@[]`/`array/push` not repeated concatenation
- [ ] Threading macros (`->`, `->>`) used where they improve readability
- [ ] Short-fn `|` syntax used for simple lambdas
- [ ] Destructuring used in function parameters where clear
- [ ] `deep=` used for structural comparison, not `=`
- [ ] `loop` uses correct verb syntax (`:range`, `:in`, `:keys`, etc.)
- [ ] Mutation scope is limited and intentional
