---
layout: post
title: "PHP2Shell And The Threat of Memory Safety Vulnerabilities"
description: "Undelying Memory-Unsafe Languages are a threat to critical web infrastructure"
date: 2026-10-27 08:00:00
---

In August 2026, researchers from [Calif](https://calif.io/) and [Wiz](https://www.wiz.io/) discovered the PHP2Shell vulnerabilities ([CVE-2026-109618](https://www.cve.org/CVERecord?id=CVE-2026-109618) and [CVE-2026-109619](https://www.cve.org/CVERecord?id=CVE-2026-109619)). They reported the findings to the PHP Foundation, and a team began engineering a patch. Both vulnerabilities were assigned a [critical CVSS v4.0 score of 9.2](https://news-web.php.net/php.announce/509). Coordinated disclosure was planned for October 22, 2026.
 
However, on October 20, 2026—two days before the planned patch date—details of the exploit were leaked in a [Hacker News thread](https://news.ycombinator.com/item?id=50167754).
 
By October 21, the Cybersecurity and Infrastructure Security Agency (CISA) issued an [urgent directive urging organizations to patch immediately](https://www.cisa.gov/news-events/alerts/2026/10/21/critical-vulnerabilities-php-require-immediate-patching). Honeypot telemetry from cloud providers lit up with active Mirai and Potassium botnet scanning, and the PHP development team accelerated out-of-band emergency releases across four active branches: PHP 8.5.12, 8.4.27, 8.3.36, and 8.2.35.
 
The PHP2Shell vulnerabilities are the direct, predictable outcome of architectural compromises made in C thirty years ago—specifically, how C programs hadle failure using `setjmp()` and `longjmp()`.
 
The PHP interpreter is powered by the [Zend Engine](https://github.com/php/php-src/tree/master/Zend), an open-source execution engine written primarily in C. C has no built-in exception handling. If PHP’s interpreter is deep inside a call stack—say, three or four functions deep while parsing a multipart POST body—and encounters an unrecoverable failure such as exceeding `memory_limit`, there is no `try`/`catch` mechanism to unwind cleanly back out to a safe point. Traditionally in C, you’d have to return an error code up through every single function in the call stack, one level at a time, and every one of those functions would need to check for it and bail out too. That’s slow to write and easy to get wrong.
 
`setjmp()` and `longjmp()` are C’s blunt workaround for this: a way to jump instantly from deep inside nested function calls back to one specific earlier point in the code, skipping all the normal return-up-the-stack steps in between.
 
- `setjmp()` is called at the designated "safe point." It saves the CPU state (stack pointer, instruction pointer, callee-saved registers) into a `jmp_buf` structure and returns `0` on the initial call.
- `longjmp()` is invoked later from any depth in the call hierarchy when an unrecoverable error occurs. It restores the saved CPU state from the buffer, causing execution to resume immediately at the `setjmp()` call site—this time returning a non-zero value indicating an abnormal abort.
 
The specific mechanics of how PHP leverages `setjmp()`/`longjmp()` are documented in detail in the [official Calif write-up by security researcher Ryan Austin](https://calif.io/php2shell/). Ryan discovered the vulnerabilities with the aid of OpenAI's GPT-5.6 Sol. If you haven't already, please read Ryan's write-up if you're interested in a technical deep dive into PHP2Shell. 

Rather than unwinding every frame sequentially, PHP `longjmp()`s directly back to the nearest bailout point. While the error handler knows an abort occurred, it cannot safely determine what intermediate heap allocations, pointers, or internal structures were left partially constructed. That blind spot is precisely what the two PHP2Shell vulnerabilities exploit.
 
PHP is among the most scrutinized codebases in open source. Why did these flaws remain dormant for years? The OOM issue was identified as [early as 2020](https://bugs.php.net/bug.php?id=78876) and again in [a 2024 security audit](https://ostif.org/wp-content/uploads/2025/04/24-07-1730-REP-V1.4_temp.pdf), but the researchers could not prove that it was an exploitable issue - so it stayed classified as a hardening issue, not a vulnerability. Ryan noted that leveraging LLMs within a domain where he already possessed deep context enabled him to accelerate exploit discovery by directing the model in the right, historically-informed direction.
 
For the past five years, the global cybersecurity community has been engaged in an intensive debate regarding memory safety.
 
Agencies including the US National Security Agency (NSA), CISA, and the White House Office of the National Cyber Director (ONCD) have published [explicit guidance urging organizations to transition away from memory-unsafe languages](https://rustify.rs/articles/rust-memory-safety-nsa-cisa-2026) (C and C++) for critical infrastructure.
 
The data supports this push. Historical analysis revealed that roughly [70% of all severe security bugs in Chromium](https://www.chromium.org/Home/chromium-security/memory-safety/) were memory safety violations. Decades of MSRC telemetry showed an identical metric: [~70% of CVEs patched annually in Microsoft](https://www.microsoft.com/en-us/msrc/blog/2019/07/we-need-a-safer-systems-programming-language) software stem from memory safety flaws. Furthermore, memory safety vulnerabilities in [Android dropped from 76% of all vulnerabilities down to under 25%](https://security.googleblog.com/2024/09/eliminating-memory-safety-vulnerabilities-Android.html) over a four-year period after Google adopted Rust for new native components.
 
The PHP core runtime represents approximately 800,000 lines of mature C code, maintained by capable systems programmers, fuzzed continuously via OSS-Fuzz, and validated by hundreds of thousands of tests. Yet, an unauthenticated remote attacker could compromise an unpatched server through a textbook use-after-free and an uninitialized memory condition.
 
Why? Because in C, the developer must maintain an airtight mental model of every memory allocation, pointer lifetime, and potential non-local jump across millions of state transitions. As codebases grow in complexity over decades, that mental model inevitably breaks down.
 
The most realistic long-term engineering path is the incremental replacement of high-risk subsystems using a memory-safe systems language such as Rust. Rather than rewriting 800,000 lines of code at once, critical components can be encapsulated behind stable C foreign-function interfaces (FFI). For instance, the memory manager ([`zend_mm`](https://github.com/php/php-src/blob/master/Zend/zend_alloc.c)) can be implemented in Rust, preventing dangling references and stale allocations by construction.
 
According to Web Technology Surveys, [PHP powers roughly 70% of all websites whose server-side programming language is known](https://w3techs.com/technologies/history_overview/programming_language). It remains the engine behind global commerce platforms, publishing networks, and enterprise applications.
 
When an unauthenticated Remote Code Execution flaw hits an application framework like Drupal or a CMS like WordPress, defenses can fall back on Web Application Firewalls, perimeter filtering, or middleware patches.
 
But when the vulnerability is inside the C runtime itself, that layer of safety disappears. A WAF cannot reliably model every variation of a malformed multipart stream designed to trip an internal memory limit—and a virtual patch cannot clean up dangling global pointers surviving in a persistent C worker thread.

PHP2Shell proved that when the runtime itself breaks, every layer of defense above it is an illusion.

Update your PHP versions immediately—and then start asking the hard question: how much longer can 70% of the internet afford to run on memory-unsafe foundations?