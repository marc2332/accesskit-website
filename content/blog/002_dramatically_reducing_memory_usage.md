+++
title = "Dramatically reducing AccessKit’s memory usage"
path = "dramatically-reducing-accesskits-memory-usage"
authors = ["Matt Campbell"]
date = 2023-02-08T16:51:25Z
+++

In our recent [status update](./looking-back-looking-forward/), we called out the use of a single large data structure for all accessible UI elements as a known potential weakness in the design. After that post, feedback from potential users of AccessKit made it clear that this design flaw was a pressing problem that blocked them from using AccessKit. One of these discussions also led us to a particularly attractive technique for solving the problem. So we decided to go ahead and do this optimization work, to unblock further adoption of AccessKit and get the inevitable incompatible API changes out of the way sooner rather than later. This post summarizes the results of this optimization, explains how we did it, and looks ahead to further potential optimizations.

## The numbers

To provide meaningful numbers, let’s look at a concrete example. Suppose we have a simple static text node with a bounding rectangle. Since the text string is allocated on the heap in both cases, we won’t consider it here, but we’ll look at the size of everything else. These numbers are for a typical 64-bit platform.

Before the recent optimization, each AccessKit node was a single structure with a size of 1,416 bytes, excluding heap-allocated strings and arrays.

Now, calculating the total size is more complicated, but the end result is more efficient. The short version is that the size of our example node is reduced by 5x or more.

Here’s the long version: The base node structure is only 32 bytes. Then we add 40 bytes for each dynamic property. Our example node has two properties, the text itself and the bounding rectangle, for a total of 80 bytes for the properties. Finally, each node has a class, which is a structure with a current size of 100 bytes. So the total size of our example node is 212 bytes, plus the heap-allocated text string, plus any overhead for the heap-allocated and reference-counted node class and property array. These latter overheads are partially platform-specific and more difficult to measure, but you get the idea; the new scheme is dramatically more memory-efficient.

Also note that, because the node class is reference-counted, it can be, and usually is, shared between nodes that have the same class. A node class consists of the node’s role, supported actions, and indices of set properties. So if we have multiple static text nodes that were constructed the same way, i.e. with the same properties set in the same order, these nodes will share a class. That means that the size of the shared node class is 100 bytes plus the overhead of heap allocation and reference counting, and the size of each static text node with the same class is only 112 bytes plus the overhead of heap-allocating and reference-counting the property array.

## What does this mean for AccessKit users?

As mentioned above, this optimization requires a breaking change to the AccessKit API. In short, the previous node structure with public fields has been replaced with two opaque structures, Node and NodeBuilder, which have methods for getting and setting the node properties. This new design is more future-proof, allowing further optimizations, so we don’t anticipate another breaking change of this magnitude. The necessary changes in an AccessKit provider (e.g. GUI toolkit) are tedious but straightforward. [egui](https://github.com/emilk/egui), the first GUI toolkit to integrate AccessKit, is already updated, thanks to the outstanding responsiveness of [Emil Ernerfelt](https://github.com/emilk), that project’s maintainer. Check out the recent [egui commit](https://github.com/emilk/egui/commit/853d49272471cc930532798840f3101ae4bca81f) to get an idea of what changes will be required in other AccessKit integrations.

## How did we get here?

In our opinion, the final, optimized design was not obvious. That’s why we started with the naive approach of using a single large structure. We knew that approach was inefficient, but we weren’t fully satisfied with other options we considered. Alternatives that we considered included:
- Multiple arrays of key-value pairs, one for each type of value, e.g. one array for strings, another for integers, and so on. This is what Chromium’s accessibility implementation 
  uses. But we weren’t satisfied with the size overhead of multiple heap-allocated, dynamic arrays (vectors).
- A single array of key-value pairs or a hash table, in which each value was a Rust enum (called a variant in other languages) allowing all possible types. This type of enum did end 
  up being a key component of the final design. But we weren’t happy with the idea of doing linear search through an array of key-value pairs (a problem that would also affect the 
  previous option), and in one of the earliest AccessKit design discussions, we were warned about the overhead of using a hash table for each node.
- Abandoning a one-size-fits-all node structure altogether in favor of a trait (called an interface in other languages) that could be implemented by each provider. This would be more 
  similar to what the platform accessibility APIs themselves define. In principle, this would allow each AccessKit provider (i.e. GUI toolkit or application) to implement an optimal 
  structure for representing each type of node that it can provide. But this approach has its own downsides. In particular, a key property of the current AccessKit design is that all 
  nodes can be easily serialized, so an accessibility tree or incremental tree update can be pushed to another process or even another machine. We haven’t yet fully explored the 
  possibilities enabled by this feature, and we didn’t want to compromise those possibilities. We were concerned that calling an opaque function through dynamic dispatch (also known 
  as a virtual function call) for each node property would make future push-based implementations much less efficient than they can be when serializing static data. So the 
  trait/interface approach wasn’t a clear win.

Then, [Federico Mena Quintero](https://viruta.org/), a founder of the GNOME desktop project who is now working on GNOME accessibility, drew our attention to an [article about how he reduced memory usage in his librsvg project](https://viruta.org/reducing-memory-consumption-in-librsvg-2.html), which is also written in Rust, and we immediately knew that this article described an approach that would also work well for AccessKit. Briefly, this approach uses two arrays: a dynamic array (vector) of property values, and a statically sized array of entries for all properties. Each entry in the second array is either an index into the first array, or a sentinel value if the property is not set. Because both librsvg and AccessKit have fewer than 256 properties, each entry in the second array can be just one byte. Thus we avoid the overhead of a hash table, but we also don’t need to do linear search whenever we fetch a property.

This insight alone would have significantly reduced our memory usage, and we did start by implementing this optimization by itself for all properties that occupied more than 1 byte in the original structure. But then we realized we could go further. First, we packed all of the flags into a single 32-bit integer. Of course, this was an obvious optimization, but we didn’t do it before because the fields of our structure were all public, we were using automatically generated serialization code, and we wanted the serialized representation of a node to be completely natural to users of higher-level languages. In the end, we were still able to meet that latter goal through a custom implementation of serialization.

Then we realized that Federico’s array of property indices is similar to [the vtable design of FlatBuffers](https://google.github.io/flatbuffers/flatbuffers_white_paper.html). And in FlatBuffers, a single vtable can be shared by multiple tables (the primary data structure in FlatBuffers), as long as those tables are part of a single buffer. We rejected FlatBuffers itself for both legal and technical reasons. First, FlatBuffers is available solely under the Apache license, which is not compatible with GPLv2, and we wanted AccessKit to remain GPLv2-compatible. But also, a FlatBuffers vtable can only be shared across tables in a single buffer, and that wouldn’t have worked for sharing property tables across multiple accessibility tree updates. Finally, each entry in a FlatBuffers vtable is 4 bytes, as opposed to the 1-byte property indices in librsvg and AccessKit. Still, we liked the idea of a shared property table and wanted to use it somehow.

Finally, we realized that a shared property table wouldn’t require anything as esoteric as arena-allocated buffers with a custom layout, as implemented by FlatBuffers; we could do it in ordinary, safe Rust, using reference-counting and a simple shared set. Then, because the idea of dynamically constructing a property table, then determining if the table could be shared with other nodes, seemed conceptually similar to [V8’s hidden classes](https://v8.dev/docs/hidden-classes), we decided to wrap the shared property table in a construct that we call a node class, which we then extended to include the node’s role and supported actions. At this point, it also made sense to extend the set of dynamic properties to include the non-boolean properties that occupied just 1 byte in the original structure, to further reduce the base size. And with that, we arrived at the final, optimized design that yields the numbers presented above.

## Can we do better?

While this design is more optimized, it’s certainly not perfect.

As mentioned above, the size of each dynamic property is 40 bytes (for a typical 64-bit platform). While we believe this is good enough for now, we’d like to see if we can do better. The current limiting factor is the size of our rectangle structure, which is 32 bytes. To get around this, we could allocate the bounding rectangle separately on the heap, but we’re reluctant to do this for such a common property. Alternatively, because the bounding rectangle property is so common, we could include it in the base structure, increasing the size of that structure to 64 bytes. That would make the optimization less impressive on paper, but may be a win overall in real-world use cases.

While this optimization is a clear win for data size, it’s a modest regression for compiled code size in some cases. The executable size of egui’s hello_world example, when compiled with egui’s release build settings on x86-64 Windows, increased by about 15 KB. However, when compiling the same example with maximum optimization for size, link-time optimization, and panic = "abort", the AccessKit node size optimization actually decreased executable size slightly. So maybe we just need to add some more inlining hints. We’d appreciate help on this from developers with more experience optimizing Rust code. In the meantime, we believe the modest executable size increase, in cases where it does happen, is a reasonable tradeoff for the dramatic reduction in memory usage.

We haven’t yet measured the effects of this optimization on speed. It stands to reason that building a property table, determining whether that table can be shared with other nodes, then looking up properties through that table, as opposed to just getting and setting fields in a simple struct, isn’t free. But the synthetic benchmarks we might devise at this point seem unlikely to reflect real-world performance. We look forward to feedback from users as they measure AccessKit’s performance in real-world applications, particularly applications with complex, data-rich user interfaces.

Finally, we’d like to explore what we might achieve by allocating an arena for each AccessKit node, or perhaps storing all nodes and their classes in a custom heap. Maybe we can reduce the overhead that comes from multiple heap allocations (and deallocations), as well as the overhead of using absolute pointers for everything, particularly on 64-bit platforms. We recently came across a [compact garbage-collected heap in C++](https://github.com/snej/smol_world) that is particularly impressive. However, we don’t want to rush into adding unsafe code in the core of AccessKit, and we have higher-priority things that we need to work on first, so this potential optimization will remain a longer-term project.

## Conclusion

Now that the memory optimization described in this post is complete, we’re confident that the design of AccessKit is fundamentally sound. We can always optimize more, and we may still add and remove some properties before freezing the AccessKit API, but we don’t anticipate any more large-scale breaking changes that will affect GUI toolkits and applications. That means this is also a good time to begin work on bindings for other languages; after all, our goals for AccessKit extend beyond the Rust ecosystem.

We’d like to again thank Federico Mena Quintero for the article that inspired our optimization work, as well as the FlatBuffers and V8 projects for indirectly inspiring our implementation of shared node classes. And, of course, we must thank everyone who has contributed to the Rust language, for empowering us to fearlessly optimize without compromising safety. We all build on each other’s work, and now that AccessKit’s design is maturing, we hope that many more developers will build on our own work to make their applications accessible.