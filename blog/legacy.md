# Legacy

## Why legacy ?
We use to think of legacy as a pejorative word. 
It can be a synonym on inheritance, of monetary gain, but we think of as debt.

From whom are we inheriting ? We used to think of former developers, but it would be better to think of features, eg. problem solved by code.
We inherit a implemented problem-solver. Our job is to add capacity to solve more problems.
Of course, we can leverage the existing code, to reuse some of it component (may it be throught copy/paste or through module).

Heuristic
- context
- constraints
- developer's SOTA

Context is everything: what we deemed negative des not apply anywhere and anytime
Thick client over http are considered obsolete, but they were a great improvement over terminal.
We may deconsider some technology we never practiced, or practiced without any help.
If we work on an application with this technology, we had better, when calling it legacy, to make our feeling explicit.


## Debt
If we use "technical debt" term to mean 
- technologies that are no longer supported (eg. Opera browsers, ALGOL compiler on some platforms)
- technologies for which developers are no longer available (technologies may be old of new: SQL training is valuable since a long time, and some front-end javascript frameworks are yound but extinct - short-lived) 

We do not focus of good/bad in abstract: the solution was fitting, but no longer fits, the tradeoff is seriously off.
A new programmation language has been created meanwhile, and it is more concise.
But we should not disguise our lack of knowledge as "the good way".

If we use "technical debt" to mean the former developers produce a wrong code structure, we should discuss.
We may lack some functional background and mistake functional complexity for technical complexity for 

Is the application designed for 
- new developers
- old developers ?

COntext is everything.

Someone who wants to kill its dog accuse him of rabe.

## What would be a good legacy ?

###  Functional

Problem domain modelization, ie. entities and behaviours, if they reflect the business implicit modelization (check it with dialog with them), is valuable.
If this can be kept in a module, so that technical considerations (eg. serialization, persistence) can be put in aniother module, you're kind of building a knowledga base.
Such techniques as hexagonal architecture and DDD aims at it.

Solution are context-dependent, the rationale in time is expressed through ADRs
