+++
title = "2. Composing objects"
date = "2025-01-15"
tags = [
    "markdown",
    "syntax",
]
+++


We don't want to have to analyze each memory access to ensure that our program is thread‐safe; we want to be able to 
take thread‐safe components and safely compose them into larger components or programs.

This chapter covers patterns for structuring classes that can make it easier to make them thread‐safe and to 
maintain them without accidentally undermining their safety guarantees.
# 1. Designing a thread-safe class

## 1.1. Gathering Synchronization Requirements
*Invariants* are defined as constraints that make a certain state of the object invalid or valid. For example, `NumberRange` instance
has two states upper and lower ranges, the lower-range value must always is lower than the upper one.

*Post-conditions* are defined as a state an object must have when it is transformed from the current state. For example, 
If the current state of a Counter is 17, the only valid next state is 18.

Your code must ensure to comply *invariants* and *post-conditions* while synchronization. If it fails to do that, the code will be left
in an inconsistent state.

For instance, as mentioned, `NumberRanger` has a multivariable invariant (by multivariable, it means the invariant involves many states
as one). You cannot update one, release and reacquire the lock, and then update the others, 
since this could involve leaving the object in an invalid state when the lock was released. 
When multiple variables participate in an invariant, the lock that guards them must be held for 
the duration of any operation that accesses the related variables.

> By using a constant field you limit the state space an object can have, which 
> is easier to deal with when it comes to synchronization.


## 1.2. State-dependent operations
An operation is said to be state-dependent when it can not be applied to the current state of an object if the object is in certain
states (*preconditions*).

For example, if you call `popElement()` when a queue is empty in a single-threading environment, it will throw error.
In multi-threading environment, however, if you dedicate a thread to wait there to `popElement` once a queue is not empty and do the task
on that element, there is chance other threads will add an element to the queue, which turn the *precondition* to `true`.

In these cases, to make thread continue to run after a condition is met, we often:
+ Use low-level mechanisms such as `wait` and `notify` (covered in chapter 14).
+ Use existing library classes, such as blocking queues or semaphores (covered in chapter 5) <- this is a more straight forward approach.