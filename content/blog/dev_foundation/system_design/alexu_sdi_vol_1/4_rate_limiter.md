+++
title = "4. Design rate limiter"
date = "2025-03-28"
tags = [
    "markdown",
    "syntax",
]
+++

Benefits of rate limiter:
+ Prevent DDOS attack.
+ Reduce cost: fewer servers, allocating resources for higher priority APIs,
fewer calls to third-party services which charge you per API call.
+ Prevent servers from being overloaded: multiple requests may be from another crawling service, or
users' misbehavior.