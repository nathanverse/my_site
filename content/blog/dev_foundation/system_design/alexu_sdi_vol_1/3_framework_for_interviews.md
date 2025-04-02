+++
title = "3. A framework for system design interviews"
date = "2025-03-28"
tags = [
    "markdown",
    "syntax",
]
+++

# The interview framework

This chapter gives insights about what you should do during the system design interview.
Four basic steps are:
1. Step 1 - Understand the problem and establish design scope: This is when you ask the interviewer questions about
the constraints of the system, or making assumption if needed. The goal is to understand the load of the system and the timeline
to shift the feature. Ask:
   + What specific features are we going to build?
   + How many users does the product have? 
   + How fast does the company anticipate to scale up? What are the anticipated scales in 3 months, 6 months, and a year? 
   + What is the company’s technology stack? What existing services you might leverage to simplify the design?
2. Step 2 -  Propose high-level design and get buy-in: develop a high-level design and reach an agreement with the interviewer on the design. 
Treat your interviewer as a teammate and work together during the process. Do back-of-the-envelope calculations (if necessary) and go through a few concrete use
cases to illustrate your design.
3. Step 3 - Design deep dive: At this step, you and the interviewers should have already sketched out a high-level blueprint for the overall design, now 
you shall work with the interviewer to identify and prioritize components in the architecture. Each interviewer may concern a different aspect, such as, bottleneck,
digging deeper in a specific component,...
4. Step 4 - Wrap up: You may be asked several follow-up questions before the interview ends, which may be:
   + Identify the system bottlenecks and discuss potential improvements.
   + Recap the design (it is also helpful if you could actively to do this after a long session).
   + Error cases (sever failure, network loss).
   + Operation issues, such as how to monitor error logs, metrics, or roll out the system.
   + How to handle the next scale curve, like 10 mil users.
   + Propose other refinements you need if you had more time.

# Time allocation
Time allocation is essential to not get yourself rambling on and on. Following is a raw estimation, the actual one 
may depend on requirements and scopes.
+ Step 1 Understand the problem and establish design scope: 3 - 10 minutes 
+ Step 2 Propose high-level design and get buy-in: 10 - 15 minutes
+ Step 3 Design deep dive: 10 - 25 minutes
+ Step 4 Wrap: 3 - 5 minutes