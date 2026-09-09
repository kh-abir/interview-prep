# 01 — Ruby Language Fundamentals (Zero to Senior)

> **Context**: Primary Backend Language (5+ years production experience). Focuses on internal VM mechanics (MRI/CRuby), memory layout, object model dispatch, and concurrency.

---

## 1. Core Syntax, Data Types & Memory Internals

### 1.1 Concept & Variable Scopes
Ruby variables are references to objects stored on the Ruby heap:
- **Local variables** (`foo`): Scoped to current lexical scope (block, method, or class body). Managed via VM stack frames.
- **Instance variables** (`@foo`): Bound to `self`. Stored directly in the object's internal struct (or overflow array if attributes > 3 in CRuby `RObject`).
- **Class variables** (`@@foo`): Shared across the inheritance hierarchy (dangerous anti-pattern: changing `@@foo` in a child class mutates the parent class and all siblings).
- **Class instance variables** (`@foo` at class level): Bound to the class object itself (cleaner encapsulation than `@@`).
- **Global variables** (`$foo`): Accessible across entire VM; bypasses lexical boundaries.
- **Constants** (`FOO`): Lexically looked up via `Module.nesting`, then ancestor chain. Dynamic mutation triggers warnings (`warning: already initialized constant`).

### 1.2 Symbols vs. Strings & Frozen String Literals
- **String (`String`)**: Mutable byte sequence. Every dynamic string allocation `str = "hello"` creates a new `RString` object on the heap.
- **Symbol (`Symbol`)**: Immutable identifier. Stored in CRuby's internal global Symbol Table (`global_symbols`). Historically leaked memory; since Ruby 2.2, dynamic symbols created via `to_sym` are garbage-collected (immortal vs mortal symbols).
- `# frozen_string_literal: true`: Magic comment placed at the top of `.rb` files. Freezes all string literals in the file into single, deduplicated, read-only heap allocations (`can't modify frozen String (FrozenError)` on mutation).

```ruby
# frozen_string_literal: true

# Benchmark: Dynamic string vs Frozen string allocation
require 'benchmark/ips'

STR = "transaction_event_id"

Benchmark.ips do |x|
  x.report("unfrozen string") do
    "transaction_event_id".dup
  end

  x.report("frozen string literal") do
    "transaction_event_id"
  end

  x.compare!
end
# Output: Frozen string literal is 10x-20x faster and causes 0 new allocations!
```

---

## 2. Blocks, Procs, and Lambdas

### 2.1 Mechanical Comparison

| Feature | Block | `Proc` (`Proc.new`) | `Lambda` (`lambda` or `->()`) |
|---|---|---|---|
| **Class** | Not an object (syntax element) | `Proc` | `Proc` (`lambda? == true`) |
| **Arity Checking** | Lenient (extra args discarded, missing set to `nil`) | Lenient | **Strict** (`ArgumentError` on mismatch) |
| **`return` Semantics** | Returns from the *enclosing method* | Returns from the *enclosing method* (breaks method stack) | Returns from the *lambda itself* |
| **Conversion** | Converted to Proc via `&block` | Coerced to block via `&proc` | Coerced to block via `&lambda` |

```ruby
# Method Return Behavior Comparison
def test_proc_return
  p = Proc.new { return "proc return" }
  p.call
  "method reached end" # NEVER REACHED: Proc exits enclosing method
end

def test_lambda_return
  l = -> { return "lambda return" }
  res = l.call
  "method reached end with #{res}" # REACHED: Lambda returns back to caller
end

puts test_proc_return   # => "proc return"
puts test_lambda_return # => "method reached end with lambda return"
```

### 2.2 Advanced Idioms: Method-to-Proc & Currying
```ruby
# & operator calls Symbol#to_proc -> &:strip.to_proc
users = ["  alice ", " bob  ", "charlie\n"]
clean_users = users.map(&:strip)

# Currying for reusable business rule pipelines
discount_calculator = ->(rate, tax, amount) { (amount * (1 - rate)) * (1 + tax) }
fintech_tax_discount = discount_calculator.curry[0.10, 0.05] # Fix 10% discount, 5% tax

final_price = fintech_tax_discount.call(1000) # => 945.0
```

---

## 3. Object Model & Metaprogramming

### 3.1 Everything is an Object & The Ancestor Chain
In CRuby, every object has a pointer to its class (`klass`). Classes themselves are instances of the `Class` class.

```
       BasicObject
            ▲
          Object
            ▲
         [Module] (prepended or included modules)
            ▲
          ParentClass
            ▲
          Subclass
            ▲
      [Eigenclass] (Singleton Class: class << obj)
            ▲
          instance (obj)
```

- **Method Lookup Order**:
  1. Object's **Eigenclass** (holds singleton methods)
  2. Modules **prepended** to the class (in reverse order of prepend)
  3. The class definition itself
  4. Modules **included** in the class (in reverse order of include)
  5. Superclass ancestor chain up to `Object` -> `Kernel` -> `BasicObject`

### 3.2 `include` vs. `extend` vs. `prepend`
- `include M`: Inserts `M` directly *above* the class in the ancestor chain (`Class < M < SuperClass`). Provides instance methods.
- `extend M`: Includes `M` into the object's *eigenclass*. When called in a class body, adds class methods.
- `prepend M`: Inserts `M` *below* the class in the ancestor chain (`M < Class < SuperClass`). Allows wrapping/decorating existing methods via `super`.

```ruby
module Auditable
  def charge!(amount)
    Rails.logger.info("Initiating charge of $#{amount}")
    result = super # Invokes original class method
    Rails.logger.info("Charge complete: #{result}")
    result
  end
end

class PaymentGateway
  prepend Auditable

  def charge!(amount)
    "Captured $#{amount}"
  end
end

PaymentGateway.ancestors
# => [Auditable, PaymentGateway, Object, Kernel, BasicObject]
```

### 3.3 Dynamic Dispatch & Method Synthesis
```ruby
class DynamicApiGateway
  # 1. define_method: Evaluates in closure context, fast, clean
  %i[get post put delete].each do |http_verb|
    define_method("execute_#{http_verb}") do |endpoint, payload = {}|
      connection.send(http_verb, endpoint, payload)
    end
  end

  # 2. method_missing + respond_to_missing?: Golden rule: Always pair them!
  def method_missing(method_name, *args, &block)
    if method_name.start_with?("find_by_")
      field = method_name.to_s.sub("find_by_", "")
      record_store.find { |record| record[field] == args.first }
    else
      super
    end
  end

  def respond_to_missing?(method_name, include_private = false)
    method_name.start_with?("find_by_") || super
  end
end
```

### 3.4 Context Evaluation: `instance_eval` vs. `class_eval`
- `instance_eval`: Evaluates code in context of *instance*. `self` becomes instance. Used for DSL builders.
- `class_eval` (alias `module_eval`): Evaluates code in context of *class*. Defines instance methods dynamically on class.

```ruby
# DSL Builder Pattern using instance_eval
class HealthCheckConfig
  def initialize(&block)
    @checks = {}
    instance_eval(&block) if block_given?
  end

  def database(url)
    @checks[:database] = url
  end

  def redis(host, port)
    @checks[:redis] = "#{host}:#{port}"
  end
end

config = HealthCheckConfig.new do
  database "postgres://localhost:5432/main_db"
  redis "127.0.0.1", 6379
end
```

---

## 4. Enumerable Mastery & Lazy Sequences

### 4.1 High-Performance Enumerable Idioms
```ruby
transactions = [
  { id: 1, account: "A", amount: 150.0, status: :completed },
  { id: 2, account: "B", amount: 80.0,  status: :failed },
  { id: 3, account: "A", amount: 220.0, status: :completed },
  { id: 4, account: "C", amount: 45.0,  status: :completed }
]

# tally: Count occurrences without boilerplate hash counting (Ruby 2.7+)
statuses = transactions.map { |t| t[:status] }.tally
# => {:completed=>3, :failed=>1}

# each_with_object: Functional transformation without mutating external accumulator
account_totals = transactions.each_with_object(Hash.new(0.0)) do |tx, acc|
  acc[tx[:account]] += tx[:amount] if tx[:status] == :completed
end
# => {"A"=>370.0, "C"=>45.0}

# partition: Split collection into match and non-match in single pass
completed, pending_or_failed = transactions.partition { |t| t[:status] == :completed }
```

### 4.2 Lazy Enumerators for Large Streams
Standard Enumerable creates intermediate arrays in memory at every step. `lazy` creates an `Enumerator::Lazy` pipeline that pulls elements one by one.

```ruby
# Reading multi-gigabyte log files without OOM
def scan_critical_errors(log_filepath)
  File.open(log_filepath)
      .lazy
      .map(&:chomp)
      .select { |line| line.include?("SEV-1") || line.include?("FATAL") }
      .map { |line| JSON.parse(line) }
      .take(10) # Stops immediately after finding 10 matches!
      .to_a
end
```

---

## 5. Concurrency in Ruby: GVL, Threads, Fibers, and Ractors

### 5.1 The GVL (Global VM Lock) Explained
- CRuby execution is protected by GVL. Only **one OS thread executes Ruby bytecodes at any given moment**.
- **Crucial Rule**: When a thread initiates an **I/O operation** (DB query, HTTP request, socket read/write), CRuby **releases the GVL**.
- Consequence: Ruby multi-threading achieves high concurrency for **I/O-bound workloads**, but cannot achieve multi-core parallelism for **CPU-bound workloads**.

```
[ Thread 1: DB Query ] ---> Releases GVL -> OS handles socket wait
                                │
[ Thread 2: Parses JSON] <--- Acquires GVL -> Executes on CPU
```

### 5.2 Concurrency Primitives Comparison

```ruby
# 1. Threads (I/O Concurrency)
threads = 5.times.map do |i|
  Thread.new do
    Net::HTTP.get(URI("https://api.internal/service/#{i}"))
  end
end
responses = threads.map(&:value)

# 2. Fibers (Cooperative Coroutines - Zero OS Context Switch Overhead)
fiber = Fiber.new do
  puts "Fiber step 1"
  Fiber.yield 42
  puts "Fiber step 2"
  100
end
val1 = fiber.resume # prints "Fiber step 1", returns 42
val2 = fiber.resume # prints "Fiber step 2", returns 100

# 3. Ractors (Ruby 3.0+ True Multi-Core Parallelism)
r1 = Ractor.new do
  sum = (1..10_000_000).reduce(:+)
  Ractor.yield sum
end

r2 = Ractor.new do
  sum = (10_000_001..20_000_000).reduce(:+)
  Ractor.yield sum
end

total = r1.take + r2.take # Both ran simultaneously on separate CPU cores!
```

---

## 6. Memory Management, ObjectSpace, and GC Tuning

### 6.1 CRuby Generational GC Internals
- **Slot Architecture**: Heap is divided into 16KB Pages containing 40-byte `RVALUE` slots.
- **Generational GC (RGenGC)**:
  - **Young / Nursery Objects**: Newly allocated. Sweep occurs frequently (minor GC).
  - **Old Generation Objects**: Promoted after surviving 3 minor GC cycles.
  - **Major GC**: Scans young + old generation. Expensive "stop-the-world" pause.

### 6.2 Production Memory Leak Hunting
```ruby
require 'memory_profiler'

report = MemoryProfiler.report do
  1_000.times do
    HeavyReportService.call(tenant_id: 12)
  end
end

report.pretty_print(to_file: "tmp/memory_leak_report.txt")
```

### 6.3 GC Tuning Variables in Production
```bash
# Production environment (Docker / ECS task definition)
export RUBY_GC_HEAP_INIT_SLOTS=1000000        # Pre-allocate slots to prevent page churn at boot
export RUBY_GC_HEAP_FREE_SLOTS=500000         # Target free slots after sweep
export RUBY_GC_HEAP_GROWTH_FACTOR=1.25        # Conservative growth to mitigate sudden RSS bloat
export RUBY_GC_MALLOC_LIMIT=64000000          # 64MB C-malloc threshold before triggering GC
```

---

## 7. Senior Interview Q&A Cheatsheet

### Q1: "Why does `str += 'a'` in a loop cause memory bloat, and what should you do instead?"
> **Answer**: `str += 'a'` is sugar for `str = str + 'a'`. Each iteration allocates a brand-new `RString` object on heap and copies all previous bytes, resulting in $O(N^2)$ memory copying and thousands of short-lived objects triggering GC pressure. Instead, use in-place mutation `str << 'a'` ($O(1)$ amortized append within pre-allocated buffer), array join `[].push('a').join`, or `StringIO`.

### Q2: "Can you achieve true multi-core parallel execution with Ruby Threads?"
> **Answer**: In CRuby (MRI), no, because of GVL (Global VM Lock). Only one thread executes Ruby VM instructions at any instant. However, during blocking I/O (sockets, DB queries, system calls), GVL is released, allowing concurrent I/O wait. For true multi-core parallel execution in Ruby, you must use either **multi-processing** (Puma clustered mode with worker forks), **Ractors** (Ruby 3+ actor-model parallelism with isolated memory and per-Ractor GVL), or run on alternative runtimes like JRuby / TruffleRuby.

### Q3: "What is the difference between `send` and `public_send`?"
> **Answer**: `send` bypasses Ruby encapsulation and can invoke `private` and `protected` methods on any object. `public_send` respects encapsulation and raises `NoMethodError` if method is private or protected. In production web applications, dynamic user inputs or route dispatches must always use `public_send` to prevent attackers from invoking internal lifecycle or meta-programming methods.
