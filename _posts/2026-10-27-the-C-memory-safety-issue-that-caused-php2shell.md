---
layout: post
title: "The C Memory Safety Issue That Caused PHP2Shell"
description: "Undelying Memory-Unsafe Languages are a threat to critical web infrastructure"
date: 2026-10-27 18:00:00
---


Last week, everything that could go wrong went wrong. On Tuesday, October 20, 2026, a Hacker News post hinted at an impending PHP announcement about two remote CVEs. Over the next 72 hours, there was panic and pandemonium. A number of websites went offline to avoid being exploited, including Dutch and Belgian websites. The Japanese Prime Minister posted on X urging everyone to apply the PHP patch.

In order to understand the PHP2Shell vulnerabilities ([CVE-2026-109618](https://www.cve.org/CVERecord?id=CVE-2026-109618) and [CVE-2026-109619](https://www.cve.org/CVERecord?id=CVE-2026-109619)), we'll have to examine how C handles exceptions.

Zend Engine, the execution engine that powers the PHP interpreter, is written in C, which does not have native exception handling. Instead of a `try`/`catch` block, C handles exceptions by returning the error code up the call stack, one frame at a time. Each frame reads the error code and terminates with the error code. It's a very tedious way of handling errors.
 
Instead of manually passing the error up the stack, Zend uses C native functions `setjmp()` and `longjmp()` to implement a `try`/`catch` block. In video game terms:
 
- [`setjmp()`](https://en.cppreference.com/c/program/setjmp) is a save point in the game; you can come back to this point in case you make mistakes.
- [`longjmp()`](https://en.cppreference.com/c/program/longjmp): when mistakes are made, this function takes you back to the `setjmp()` save point.

This is a very simplified explanation. The specific mechanics of how PHP leverages `setjmp()`/`longjmp()` are documented in detail in the [official Calif write-up by security researcher Ryan Austin](https://calif.io/php2shell/). Ryan discovered the vulnerabilities with the aid of OpenAI's GPT-5.6 Sol. If you haven't already, please read Ryan's write-up if you're interested in a technical deep dive into PHP2Shell. 

Back to the simplified explanation. When the PHP interpreter `longjmp()`s to the last save point, `setjmp()`, it knows an abort happened, but it cannot track the resources (memory allocation, file opens, mutex locks, etc) that were constructed between the save point and the abort point. This resource leak is the base of the two PHP2Shell exploits.
 
The OOM issue was identified as [early as 2020](https://bugs.php.net/bug.php?id=78876) and again in [a 2024 security audit](https://ostif.org/wp-content/uploads/2025/04/24-07-1730-REP-V1.4_temp.pdf), but the researchers could not prove that it was an exploitable issue - so it stayed classified as a hardening issue, not a vulnerability.
 
So, at the heart of it, this is a C memory safety issue. These can be difficult to find, even harder to exploit. With the recent advances in AI models, that gap has closed significantly. Ryan Austin states in his post:

> I focused on something I had a background in, knew its vulnerability history well, and held a theory that specific bugs lurk within the codebase. This shows that "know your target" is still as applicable as ever, even in the LLM dominated VR age. No model is simply capable of unearthing "all bugs" with simple prompts. At least for now.
 
[Android's vulnerabilities dropped significantly](https://security.googleblog.com/2024/09/eliminating-memory-safety-vulnerabilities-Android.html) after Google switched from C++ to Rust, a memory-safe language. Because [PHP powers roughly 70% of the web](https://w3techs.com/technologies/history_overview/programming_language), it should be a priority to switch to a memory-safe core.