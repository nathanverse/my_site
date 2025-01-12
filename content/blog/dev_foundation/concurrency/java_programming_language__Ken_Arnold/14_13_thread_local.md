+++
title = "9. ThreadLocal type"
date = "2025-01-12"
tags = [
    "markdown",
    "syntax",
]
+++

`ThreadLocal` allows you to store a variable that would be unique on each thread, and hence, writing
or reading on that variable need not be synchronized.

It comes with several dangers:
1. Methods across classes dependent on global states in `ThreadLocal`.
2. In case a thread is reused, such as that of a thread pool, the initial state of the thread might
not be clean, causing unexpected behaviours for the code.