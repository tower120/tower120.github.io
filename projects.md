---
layout: page
title: Projects
sidebar_link: true
---

My C++ projects in public domain (MIT license):

* **[reactive](https://github.com/tower120/reactive)** - simple, non intrusive reactive programming library for C++. (Events + Observable Properties + Reactive Properties).
* **[trackable_ptr](https://github.com/tower120/trackable_ptr)** - Smart pointer for any movable objects. When trackable object moved/destroyed, trackers updated with new object's pointer. Very powerfull thing, specially in conjuction with continuous data storage containers, like `std::vector`.
* **[SyncedChunkedArray](https://github.com/tower120/SyncedChunkedArray)** - semi-experimental implementation of container with thread-safe elements access (you may edit container elements from multiple threads), without overhead. Very high performance (literally comparable to `std::vector`). Theoretically can be ordered.
* **[threading](https://github.com/tower120/threading)** - my crossplatform, portable threading library. Mainly spinlocks, and helpers. *[#project_threading]({{ site.baseurl }}tags.html#project-threading)*
