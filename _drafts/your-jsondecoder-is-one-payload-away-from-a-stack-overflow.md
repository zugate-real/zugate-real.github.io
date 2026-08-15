---
layout: post
title: "Your JSONDecoder is one payload away from a stack overflow"
reading_time: "9 min read"
---

We had a crash that looked like memory corruption. `EXC_BAD_ACCESS`, hundreds
of identical `unbox` frames in the stack trace, only on some payloads, never
reproducible on a developer's machine.

It wasn't memory corruption. It was `JSONDecoder` running out of stack — and
once we understood why, we found not one but **three independent depth
ceilings** in Foundation that can take your app down on deeply nested JSON.
None of them are documented. Here's each one, with a repro you can run today.

## The setup

Our app renders server-driven UI: the backend sends a tree of widget
descriptions, the client decodes it into `Decodable` models and renders. Trees
nest — containers inside containers inside tabs inside cards. Most payloads
nest under 20 levels. Then one response came back with a few hundred, and the
app died inside `init(from:)`.

## Ceiling #1: JSONDecoder recurses — and your thread's stack is smaller than you think

`JSONDecoder` walks nested containers recursively: every nesting level costs a
cluster of stack frames (`unbox`, `decode`, your model's `init(from:)`, and
friends).

That's fine on the main thread, which gets **8 MB** of stack. But you don't
decode on the main thread. You decode wherever your networking library
delivers the response — and GCD worker threads get **512 KB** by default.
Swift concurrency's cooperative pool threads are similarly small.

So the same payload that decodes fine in a unit test (main thread, 8 MB)
crashes in production (worker thread, 512 KB). That's why we could never
reproduce it locally: the repro *depended on which thread ran the decode*.

Try it:

```swift
struct Node: Decodable {
    let child: Node?
}

func nested(_ depth: Int) -> Data {
    let open = String(repeating: #"{"child":"#, count: depth)
    let close = String(repeating: "}", count: depth)
    return Data((open + "null" + close).utf8)
}

// Fine on the main thread…
let payload = nested(2_000)
_ = try? JSONDecoder().decode(Node.self, from: payload)

// …crashes on a default-stack GCD worker.
DispatchQueue.global().async {
    _ = try? JSONDecoder().decode(Node.self, from: payload)  // 💥
}
```

Tune the depth for your device and model complexity — the more properties and
containers your real models touch per level, the fewer levels you get. With
our production widget models, the budget on a 512 KB stack worked out to a few
hundred levels. Not thousands. Hundreds.

## Ceiling #2: JSONSerialization caps out around 256 levels

Even before your models decode anything, the parse itself has a limit.
`JSONSerialization` — which sits under a lot of JSON handling — refuses
payloads nested beyond roughly **256 levels** and throws
`NSCocoaErrorDomain 3840`:

```swift
let deep = nested(300)
do {
    _ = try JSONSerialization.jsonObject(with: deep)
} catch {
    print(error)  // Error Domain=NSCocoaErrorDomain Code=3840
                  // "Too many nested arrays or dictionaries…"
}
```

This one at least fails with an error instead of a crash. But if your code
treats a parse failure as "malformed payload," a deeply nested but perfectly
valid response gets silently dropped — which for us meant a blank screen and
no crash report at all. Arguably worse.

## Ceiling #3: the one that really surprised us — ARC can overflow on *deallocation*

Suppose you get past both ceilings and successfully build a deep object graph.
You're not safe yet. When the graph is released, ARC tears it down
recursively: releasing the root releases its child, which releases *its*
child, and each level is another stack frame — inside the runtime, with no
`catch` and no warning.

```swift
final class LinkedNode {
    var next: LinkedNode?
}

var head: LinkedNode? = LinkedNode()
var current = head!
for _ in 0..<50_000 {
    let node = LinkedNode()
    current.next = node
    current = node
}

head = nil  // 💥 stack overflow in objc_release / swift_release
```

In our measurements the teardown overflowed at roughly **15–20k levels** —
much deeper than ceiling #1, but the crash is nastier: it happens *after* your
decode succeeded, at whatever later moment the last reference goes away, on
whatever thread happens to drop it. The stack trace points at a release deep
in the runtime, nowhere near your decoding code. Good luck correlating that
with the payload that caused it.

(This is the classic recursive-teardown problem — linked-list libraries have
dealt with it forever — but nobody thinks about it for *decoded JSON models*.)

## What we did about it

The short version of our fix, conceptually:

**Parse iteratively.** We wrote a JSON scanner that uses an explicit stack on
the heap instead of the call stack. Depth then costs heap bytes, not stack
frames — the iterative parser handles 50,000+ levels without breaking a sweat,
and a `maxDepth`/`maxNodeCount` config guards against genuinely hostile
payloads.

**Flatten before `Decodable`, not instead of it.** We didn't want to
re-implement decoding for dozens of existing model types. So a preprocessing
pass walks the tree breadth-first, decodes leaf-to-root, and replaces each
already-built subtree with a small reference marker — so by the time any
model's `init(from:)` runs, it only ever sees a *shallow* payload. No existing
`Decodable` conformance changed.

**Tear down flat, too.** Built containers are registered in a flat registry
and emptied in flat order, so ARC never walks a deep chain on release
(verified well past 100k levels).

**And if you need a fix today:** the cheapest mitigation is to run risky
decodes on a thread with a big stack. `Thread` lets you set `stackSize`
explicitly — spin one up with 8 MB, decode there, and both decode-time
ceilings move out of practical reach. Unused stack pages cost nothing thanks
to virtual memory. It doesn't help with ceiling #3, and a thread-per-decode
has its own costs under burst traffic, so treat it as a bridge, not the
destination.

We've been running the iterative approach in production behind a remote kill
switch, and we're planning to open-source the scanner once the API settles.
That'll be its own post.

## Takeaways

Foundation's JSON stack has three undocumented depth limits, and they fail
differently: `JSONDecoder` **crashes** at a few hundred levels on worker
threads, `JSONSerialization` **errors** at ~256, and ARC **crashes later, in
teardown**, at ~15–20k. If any part of your payload's shape is controlled by a
server — server-driven UI, CMS content, anything user-generated that nests —
your decode path has a ceiling you probably haven't measured. Measure it on a
**512 KB stack**, not in a unit test.

*Found an inaccuracy, or hit these limits at different depths? I'd genuinely
like to hear about it — email or LinkedIn below.*
