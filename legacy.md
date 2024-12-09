# Legacy code for the manager 
These are side notes about Christophe Thibault "Legacy - Isolated test" blog posts that are bot yet published on OCTO blog.

The aim of these posts is to give to a manager the key points about legacy:
- what is it ?
- why/how does it work ?
- when/why would it would stop working ?
- how to get out of it ?

The key point is that isolated test goodness is hidden to the manager.
Without isolated tests, everything may appear to work as usual.

These blogs post are aiming to show: 
- the power of isolated tests;
- the limits of no test/or only integrated tests.

## What is legacy code ?
Legacy mean non-testable code.
That can occur even when actual automated checks (test code) are executed in a CI/CD.
Testable code mean you can cover most of the scenario without using up energy.

You've got a legacy code when:
- all tests are manual (non-automated) tests;
- test suite only contains integrated tests.

You don't have a legacy codebase (how to name it ? codebase under test) when:
- test suite mostly contains isolated tests;
- test suite also contains some integrated test;
- some manual test (exploratory ?) are executed during review process.

Testability doesn't only rely on automated test artifact.
Superficially, if the language doesn't provide testing primitive (dependency injection, testing framework), you're obviously stuck.
What we're talking about here is more subtle: if the code under test is not modular, you'll have to write integration tests instead of isolated test, which are most costly, so you'll write less of them.
This can also apply to a language with no testing primitive: end-to-end test, exercising the application interface (REST API or GUI), suffer the same problem.

Integration test are a pain because of:
- high execution time, as out-of-process (or out-of-server) communication should be made;
- test data setup, which are complex and involving persistence (database, filesystem, cache), mostly out-of-process operations;
- if not in end-to-end test, mocking logic wich is tedious and may not reproduce the actual behaviour.  

What we can aim for:
- new feature behave as expected
- no regression in existing feature
- [code coverage](https://en.wikipedia.org/wiki/Code_coverage), which didn't guarantee correctness (assertion coverage, probed by mutation test)

## Why does it work

The motto is :
- easy to describe;
- easy to implement;
- difficult to test.

Implementation is easy because you can always write some code that behave according to specifications.

It may not work on the first delivery, it may involve numerous fixup after manual tests.
But you can, ultimately, always write some code that will satisfy the feature requirements.
Starting form a blank page is the easy way, if the feature is big enough and is not constrained by the existing architecture.

If you have to change an actual behaviour and blank page is not an option, you can use 2 workarounds:
- copy/paste: duplicate the code involved and add a conditional to separate the two tracks;
- hack: add some code, out of the existing component, that will alter the data output after the process took place.

These solutions work around modularity: instead of generalizing some code, so it can change part of its behaviour,
you keep it as is. Abstraction required by modularity is not an easy business, and without tests you cannot be sure
how accurate is you mental model. If you have no way to have some feedback on how your change break things, the easy
solution is not to change the existing code, as you know it is actually working.

It may sound counter-intuitive that good modularity, simple and clear solutions, cannot be achieved straightly.
It's much easier, when faced with multiple exception to a rule, to add a conditional instead of rising one step more in abstraction.
I's not always wise to go abstract, but when it's wise to do it, it would be foolish to do it without test's help.

A case-in-point can be found in Kent Beck's "TDD by example" introduction
> Early one Friday, the boss came to Ward Cunningham to introduce him to Peter, a prospective customer for WyCash, 
> the bond portfolio management system the company was selling. Peter said "I'm very impressed with the functionality I see.
> However, I notice you only handle U.S. dollar denominated bonds. I'm starting a new bond fund, and my strategy require that
> I handle bonds in different currencies." The boss turned to Ward, "Well, can we do it?" 

Long story short: thanks to refactoring made in the last six months, which split up a core class into different responsibilities,
the problem was much easier to ponder about.
> A quick experiment showed that iw as possible to compute with a generic Currency instead of a Dollar.
> The trick was now to make space for the new functionality without breaking anything that already worked. 
> What would happen if Ward just ran the test ? After the addition of a few unimplemented operations, 
> the bulk of the test passed. By the end of the day, all of the tests passed.
> Ward checked the code into the build and went to the boss. "We can do it" he said confidently.


## When/why would it would stop working ?

Several points are required legacy codebase to keep living:
- numerous regression and bug fixes are acceptable in production;
- developer's turnaround is low (they know the system twist and bends);
- enough time is allowed for feature delivery.

I found that legacy codebase are to be found in internal corporation application, as external business constraint
may have less impact than on the company's marketplace.

If some of these conditions change, the tradeoff may not work anymore.


## How to get out of it ?

End-to-end to tests are attractive, but;
- they cannot cover all business rules, because of combinatorial explosion and setup cost;
- they do not put any constraint of the code, so the situation will go worse. 

Modularized code covers functional decomposition, it is orthogonal to other aspects, eg in poker:
- UI
- card shuffling and distribution
- player order
- card hand
- cash flow
