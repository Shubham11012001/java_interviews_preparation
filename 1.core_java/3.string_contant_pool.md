# String Constant Pool in Java

## 1. One-line definition + why it exists

**Definition:** The String Pool is a special area in the JVM where Java keeps **one shared copy of each distinct string literal**, so the same text is not stored again and again.

**Problem it solves:** Programs use the same strings constantly: `"USD"`, `"SUCCESS"`, `"userId"`. If every use created a new object, memory would fill with duplicates, and the GC would have more work.

**What breaks without it:** Memory use goes up and object creation goes up. Comparing strings would also need a full character-by-character check every time, because you could never say "same text means same object".

---

## 2. Mental model

**Analogy:** Think of a college library with **one copy of each textbook**. When 100 students ask for "Data Structures", the librarian hands each the same copy instead of buying 100 books. This is only safe because **nobody can write in the book**. That is **String immutability**. If strings could be changed, one student's scribble would show up for everyone.

**Flow in text (Java 7 and later):**

```
Your code:  String a = "java";
            String b = "java";
            String c = new String("java");

HEAP
┌────────────────────────────────────────────┐
│  STRING POOL (inside heap)                 │
│   ┌────────┐                               │
│   │ "java" │ <──── a  (same object)        │
│   └────────┘ <──── b  (same object)        │
│                                            │
│  Normal heap area                          │
│   ┌────────┐                               │
│   │ "java" │ <──── c  (separate copy)      │
│   └────────┘                               │
└────────────────────────────────────────────┘

a == b  → true    (same object)
a == c  → false   (different objects)
a.equals(c) → true (same text)
```

---

## 3. Internals

**Three different things get confused. Keep them apart:**

| Name | What it is | Where |
|---|---|---|
| Class file constant pool | Table inside the `.class` file listing literals, class names, method names | On disk |
| Runtime constant pool | The loaded version of that table, one per class | Metaspace (Java 8+) |
| **String Pool** | One global table of actual `String` objects, one per JVM | **Heap** (Java 7+) |

**How a literal becomes a pool entry:**
1. The compiler writes `"java"` into the class file as a text entry.
2. When the JVM first runs the bytecode instruction `ldc "java"`, it **resolves** the literal. This is lazy, so it happens at first use, not at class load.
3. The JVM looks in the string pool. If `"java"` is there, it reuses it. If not, it creates the String, adds it, and returns it.

**Data structure:** The pool is a native hash table in HotSpot (called `StringTable`) with buckets and chaining, so strings that hash to the same bucket sit in a linked list. The number of buckets is fixed unless the JVM resizes it, so it matters for speed.

**Version differences (interviewers love these):**

| Version | Pool location | Notes |
|---|---|---|
| Java 6 and earlier | **PermGen** (fixed size) | Too much `intern()` gave `OutOfMemoryError: PermGen space` |
| Java 7 | **Heap** | Pool strings can now be garbage collected normally |
| Java 8+ | Heap (PermGen removed, Metaspace added) | Default bucket count about 60,013 |
| Java 11+ | Heap | Default about 65,536, and the table can resize (flag: I'm not 100% sure of exact numbers per release, so don't quote them as facts) |

You can set the bucket count with `-XX:StringTableSize=N`.

**`intern()` method:**
- If the text is already in the pool, it returns the pool's object.
- Java 6: if it is not there, it **copies** the string into PermGen.
- Java 7+: if it is not there, it stores a **reference to your own heap object**. No copy is made.

**Compile-time folding:**
```java
String x = "ja" + "va";        // compiler turns this into "java" → pool object
final String p = "ja";
String y = p + "va";           // p is a constant → also folded → pool object
String q = "ja";
String z = q + "va";           // q is NOT final → built at runtime → new heap object
```
Since Java 9, runtime concatenation uses `invokedynamic` with `StringConcatFactory`, not `StringBuilder` bytecode. This rarely comes up, but it is good to know.

**The classic puzzle:**
```java
String s = new String("a") + new String("b"); // "ab" exists only on heap, not pool
s.intern();                                    // Java 7+: pool now points to s itself
System.out.println(s == "ab");                 // Java 7+: true | Java 6: false
```
Caveat: this only prints `true` if the literal `"ab"` was not already resolved earlier in the program.

---

## 4. Alternatives & comparison

| Approach | What it does | When it wins |
|---|---|---|
| **Literal** `"abc"` | Automatic pool use | Constants, fixed known values. The default choice. |
| **`new String("abc")`** | Always makes a new heap object | Almost never. Rare cases like protecting a small substring in very old Java. |
| **`intern()`** | Manually puts a runtime string into the pool | A small set of repeated values built at runtime (e.g. country codes from a file) |
| **G1 String Deduplication** (`-XX:+UseStringDeduplication`) | GC shares the internal character array of equal strings. It does **not** merge the String objects. | Big heaps with many duplicate strings where you don't want to change code |
| **Guava `Interners.newWeakInterner()` or own map** | Your own controlled cache of canonical strings | When you need to bound or control what gets shared, and don't want to fill the JVM-wide pool |

**Decision rule:** Literals are free and automatic. For runtime duplicates, prefer dedup or your own interner over raw `intern()`.

---

## 5. Trade-offs & limitations

**Guarantees:**
- Two identical literals in the same JVM are always the same object.
- Compile-time constant expressions are folded and pooled.
- `intern()` always returns the canonical object for that text.

**Does NOT guarantee:**
- Strings made at runtime (`+` with variables, `substring`, `StringBuilder.toString()`, reading from file/DB/network) are **not** pooled automatically.
- `==` is never a safe replacement for `equals()` in real logic.
- Pool strings are not immortal. In Java 7+ they can be garbage collected when nothing references them.

**Costs:**
- `intern()` is **slower than a normal HashMap lookup** because it goes through a JVM-wide table, and a badly sized table means long chains.
- Interning millions of unique values fills the table and makes every later `intern()` and GC scan slower.
- Pool use doesn't save memory if the strings are mostly unique.

**Edge cases:**
- Literals are shared across the whole JVM, including libraries and other code in the same application server. That creates the `synchronized("lock")` danger (see section 7).
- Compile-time folding only works for **constant** expressions. Dropping `final` changes the result.

---

## 6. Real-world production scenarios

**1. Finance/analytics SaaS (like credit risk reporting)**
- **Problem:** You load 30 million rows of company data. Columns like `currency`, `country`, `risk_band` have only a few dozen distinct values, but every row creates its own String. Heap usage is huge.
- **Choice:** Use G1 String Deduplication, or a small custom interner for those specific columns.
- **Why:** It cuts memory a lot with no change to business logic.
- **Caveat:** Never intern high-cardinality columns like company IDs or names. You'd just bloat the table.

**2. Open source: Jackson JSON parser**
- **Problem:** Parsing millions of JSON objects produces the same field names (`"id"`, `"amount"`) again and again.
- **Choice:** Jackson keeps its own symbol table and a cache for canonical names. There is a feature for interning field names (`JsonFactory.Feature.INTERN_FIELD_NAMES`).
- **Why:** Repeated keys share one object, which saves memory and makes key comparison faster.
- **Caveat:** Only field names are canonicalized, not values. Exact defaults changed across Jackson versions, so check the docs for your version.

**3. Legacy migration from Java 6 to Java 8**
- **Problem:** An old system called `intern()` on many values and kept hitting `PermGen space` errors.
- **Choice:** Upgrade to Java 8.
- **Why:** The pool moved to the heap and became collectable, and PermGen was removed.
- **Caveat:** The fix is not permanent. Interning unbounded data can still hurt through heap pressure and slow pool lookups.

**4. Bug: a shared lock hidden in a library**
- **Problem:** Two unrelated modules both wrote `synchronized ("LOCK")`. They accidentally shared one lock, so threads blocked each other.
- **Choice:** Use `private final Object lock = new Object();`.
- **Why:** That object is unique to your class, so no other code can share it.
- **Caveat:** This bug is nearly impossible to find from code review alone. You find it from thread dumps.

**5. Bug: `==` works in tests but fails in production**
- **Problem:** `if (status == "ACTIVE")` passed in unit tests, where the value was a literal, and failed in production, where the value came from the database.
- **Choice:** Use `"ACTIVE".equals(status)` or an enum.
- **Why:** DB and network strings are fresh heap objects, not pool strings.
- **Caveat:** Static analysis tools like SonarQube flag this pattern, so turn those rules on.

---

## 7. Common mistakes & failure modes

| Mistake | What happens | Detect | Fix |
|---|---|---|---|
| Using `==` on strings | Random wrong results | Code review, SonarQube | Use `equals()`, or `Objects.equals()` when null is possible |
| `intern()` on unbounded user input | Pool grows, slow lookups, memory pressure | `jcmd <pid> VM.stringtable`, heap dump | Stop interning. Use a bounded cache or dedup. |
| `synchronized` on a literal | Hidden shared lock, deadlocks or slowness | Thread dump | Use a private lock object |
| `new String("abc")` everywhere | Wasted objects | Heap histogram (`jmap -histo` or `jcmd GC.class_histogram`) | Use literals |
| Not checking duplicates | High heap use with many equal strings | Eclipse MAT "Duplicate Strings", or Java Flight Recorder | Enable G1 dedup, or canonicalize at load time |
| Long pool bucket chains | Slow `intern()` | `-XX:+PrintStringTableStatistics` (Java 8), or `jcmd VM.stringtable` | Raise `-XX:StringTableSize` |

---

## 8. Code

**Minimal correct example:**
```java
public class PoolDemo {
    public static void main(String[] args) {
        String a = "java";
        String b = "java";
        String c = new String("java");
        String d = c.intern();

        System.out.println(a == b);       // true  - same pool object
        System.out.println(a == c);       // false - c is a separate heap object
        System.out.println(a == d);       // true  - intern returns the pool object
        System.out.println(a.equals(c));  // true  - same text
    }
}
```

**Buggy vs fixed: comparing strings**
```java
// BUGGY
String status = rs.getString("status");   // fresh heap object from DB
if (status == "ACTIVE") { ... }           // often false!

// FIXED
if ("ACTIVE".equals(status)) { ... }      // null-safe and correct
```

**Buggy vs fixed: locking**
```java
// BUGGY - shared with any other code that uses the same literal
synchronized ("LOCK") { ... }

// FIXED
private final Object lock = new Object();
synchronized (lock) { ... }
```

**Buggy vs fixed: interning user data**
```java
// BUGGY - unbounded growth
String key = request.getParameter("sessionToken").intern();

// FIXED - only repeated, low-cardinality values are worth sharing
private static final Interner<String> INTERNER = Interners.newWeakInterner(); // Guava
String country = INTERNER.intern(row.getCountry());
```

---

## 9. Interview questions

### Basic (5)

**B1. What is the String Pool?**
Scenario: your app creates the same string thousands of times. Problem: that wastes memory. Choice: Java keeps one shared object per literal in the String Pool. Reason: strings are immutable, so sharing is safe. Caveat: only literals and constant expressions are pooled automatically, and runtime-built strings are not.

**B2. Difference between `==` and `equals()` for strings?**
Scenario: you check a status value. Problem: `==` compares references, not text. Choice: use `equals()` for content. Reason: two different objects can hold the same text, and `==` would say false. Caveat: `==` can look like it works for literals, which hides the bug until production.

**B3. How many objects does `new String("hello")` create?**
Scenario: the interviewer shows this line. Problem: people say "always 2" or "always 1". Choice: say "up to two". Reason: one is the pool literal, created only if `"hello"` isn't already in the pool, and one is the new heap copy. Caveat: if the literal was already pooled, only one new object is created.

**B4. Why is String immutable?**
Scenario: many parts of a program share one String. Problem: if one part could change it, everyone would see the change. Choice: Java makes String immutable. Reason: this makes pooling, thread safety, cached `hashCode`, and secure use as map keys and class names possible. Caveat: the cost is that every modification creates a new object.

**B5. What does `intern()` do?**
Scenario: you have a string built at runtime. Problem: it is not in the pool, so `==` with the literal fails. Choice: call `intern()`. Reason: it returns the pool's canonical object, adding yours if needed. Caveat: it is slower than a normal cache and should be used rarely.

### Intermediate (5)

**I1. Where was the pool in Java 6 vs 7+?**
Scenario: an old system hit `PermGen space` errors. Problem: the pool lived in PermGen, which had a fixed size. Choice: Java 7 moved it to the heap. Reason: the heap is much larger and its strings can be garbage collected. Caveat: Java 8 then removed PermGen entirely, so the error name changed to `Metaspace`, which is about class metadata.

**I2. Is `"a" + "b" == "ab"` true?**
Scenario: concatenating two literals. Problem: people think concatenation always makes a new object. Choice: answer true. Reason: the compiler folds constant expressions into `"ab"` at compile time. Caveat: if either side is a non-final variable, it becomes a runtime concat and the result is false.

**I3. Why can `==` sometimes work on strings?**
Scenario: a junior says "it worked in my test". Problem: the test used only literals, which share pool objects. Choice: explain that `==` accidentally works for pooled references. Reason: same literal means same object. Caveat: strings from DB, network, `StringBuilder`, or `substring` are not pooled, so the code breaks later.

**I4. `String.intern()` vs G1 String Deduplication?**
Scenario: a huge heap full of repeated strings. Problem: you want to save memory. Choice: `intern()` changes which object your code points to, while dedup is done by the GC. Reason: dedup merges only the internal character or byte array, not the String object, and needs no code change. Caveat: dedup works only with G1 by default (other collectors added support in later versions, so check), and it happens in the background, not instantly.

**I5. Can pooled strings be garbage collected?**
Scenario: someone claims the pool leaks memory forever. Problem: that was nearly true in Java 6. Choice: say in Java 7+, yes they can be collected. Reason: the pool moved to the heap and holds entries weakly. Caveat: a string referenced from anywhere (for example by a static field) stays alive.

### Scenario: "when would you use..." (5)

**S1. When would you use `intern()`?**
Scenario: you load millions of records with a few dozen distinct country codes. Problem: a separate String per row wastes memory. Choice: intern or canonicalize only that low-cardinality column. Reason: lots of duplicates and few distinct values gives a big saving. Caveat: measure first, and consider a custom interner or G1 dedup to avoid stressing the global pool.

**S2. When would you avoid `intern()`?**
Scenario: a service takes session tokens and request IDs. Problem: every value is unique, so interning saves nothing. Choice: don't intern. Reason: the pool fills up, bucket chains grow, and every later `intern()` and GC pause gets worse. Caveat: this problem doesn't crash the app, it just slows it down, so it is easy to miss.

**S3. When would you use `new String(...)`?**
Scenario: almost never today. Problem: in very old Java versions, `substring` kept a large backing array alive. Choice: people wrapped it in `new String(...)` to get a copy. Reason: the copy freed the big array. Caveat: Java 7u6+ changed `substring` to copy, so this trick is obsolete.

**S4. Lock choice: is `synchronized("LOCK")` okay?**
Scenario: two libraries in the same JVM both lock on `"LOCK"`. Problem: they share one lock object because literals are pooled. Choice: use `new Object()` as a private lock. Reason: no other code can get a reference to it. Caveat: this bug only shows up under load, so look for it in thread dumps.

**S5. Memory is high in production. How do you check if duplicate strings are the cause?**
Scenario: heap grows and GC gets slower. Problem: you suspect repeated strings. Choice: take a heap dump and open Eclipse MAT's duplicate-strings view. Reason: it shows which strings repeat and who holds them. Caveat: before changing code, try G1 dedup with `-XX:+UseStringDeduplication` and measure, since it needs no code change.

### Tricky follow-ups (3)

**T1. `String s = new String("a") + new String("b"); s.intern(); System.out.println(s == "ab");` What prints?**
Scenario: the classic trap. Problem: people answer from Java 6 rules. Choice: answer true on Java 7+. Reason: `"ab"` was never in the pool, so `intern()` stores the reference to `s` itself, and the later literal finds it. Caveat: on Java 6 it was false because `intern()` copied the string into PermGen, and it is also false if `"ab"` was already pooled earlier.

**T2. `final String a = "x"; String b = a + "y"; b == "xy"?`**
Scenario: a variable is involved but the answer is still true. Problem: people assume any variable forces a runtime concat. Choice: true. Reason: `a` is a final variable with a constant value, so the compiler treats `a + "y"` as a constant expression and folds it. Caveat: remove `final`, or assign `a` from a method call, and it becomes false.

**T3. Is the string pool the same as the class constant pool?**
Scenario: the interviewer tests whether you mix up two tables. Problem: the names sound alike. Choice: say no, they are different. Reason: the constant pool is a per-class table in the `.class` file and runtime metadata, holding text entries, while the string pool is a single JVM-wide table of real `String` objects on the heap. Caveat: a literal links the two, because when `ldc` runs it resolves a class-pool entry into a string-pool object.

---

## 10. Cheat sheet

- One shared object per literal, stored in the heap (Java 7+), inside `StringTable` (a hash table).
- Literals and compile-time constants are pooled. Runtime-built strings are not.
- `==` compares references and `equals()` compares text. Always use `equals()`.
- `new String("x")` makes a separate heap object.
- `intern()` returns the pool object. In Java 7+ it stores your reference if the text isn't there.
- Java 6 used PermGen (fixed size, caused OOM). Java 7+ uses heap. Java 8 removed PermGen.
- `final` constant variables get folded, but non-final ones do not.
- G1 dedup shares the backing array, not the String object.
- Never `synchronized` on a literal.
- Tune with `-XX:StringTableSize`. Inspect with `jcmd <pid> VM.stringtable`.

**30-second spoken pitch:**
"The String Pool is a JVM-wide table on the heap, since Java 7, that stores one shared copy of each string literal. This saves memory and is safe only because strings are immutable. Literals and compile-time constants go in automatically, but strings built at runtime, like from a database or a StringBuilder, do not. That's why `==` is unreliable and I always use `equals()`. I can call `intern()` to canonicalize repeated low-cardinality values, but I avoid it for unique data, because it is slower than a normal cache and can bloat the pool. For big heaps with many duplicates, I'd first try G1 String Deduplication. And I never synchronize on a string literal, because it is shared across the whole JVM."

---

## 11. Self-test

I'll ask one question at a time, wait for your answer, and then critique it. Speak it out loud first, in the format **Scenario → Problem → Choice → Reason → Caveat**, then type it.

**Question 1 of 5:**

```java
String a = "java";
String b = new String("java");
String c = b.intern();

System.out.println(a == b);
System.out.println(a == c);
System.out.println(b == c);
```

What does each line print, and **how many String objects exist after these three lines** (assume `"java"` wasn't used anywhere else before)? Explain why.