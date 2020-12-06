# TL,DR
Today's automated testing technique state-of-the-art properly address database concerns.
However, in a few corner cases, you'll need regression tests that would be a chore to write.
I point out several techniques that can help you to do that.

# Are database tests worth talking about ?
Automated testing started mostly in GUI applications.
When such application involved a split between client a server (like SPA over REST API), automated testing
didn't change much. The Subject under test (SUT) was still a general-purpose programing 
language component set, executed in various isolation degree, in memory.

Writing a test framework is not an easy task, but testing side effects takes you at another level.  
Database had his own share here, being the major side effect in most application.
[Many techniques](https://martinfowler.com/bliki/InMemoryTestDatabase.html) were designed to both:
- keep execution time low
- avoid false negatives (there's a bug, but the tests pass).

I think such techniques are successful, but there's still a blind spot somewhere.
When you have SQL queries such as this, everything is under control. 
````sql
UPDATE users SET email=<SOME_EMAIL> WHERE id=<SOME_ID>; 
````
You can neatly check the email value after the query took place.

But what about [this one](https://blogs.oracle.com/sql/obfuscated-sql-contest-winners) ?
````sql
SELECT SUBSTR(s,INSTR(s,'<S>')+3,INSTR(s,'</S>')-INSTR(s,'<S>')-3)
FROM ( SELECT DBMS_XMLGEN.getxmltype(LISTAGG(CHR((INSTR(S,b,1,3*LEVEL-2)-
DECODE(LEVEL,1,0,INSTR(S,b,1,3*LEVEL-3))-1)*25+(INSTR(S,b,1,3*LEVEL-1)-INSTR(S,b,1,3*LEVEL-2)-1)*5+INSTR(S,b,1,3*LEVEL)-INSTR(S,b,1,3*LEVEL-1)-1+32)) 
WITHIN GROUP(ORDER BY LEVEL)) S FROM ( SELECT '                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     ' ||'                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      ' ||'                                                                       ' S,chr(9) b FROM DUAL ) CONNECT BY LEVEL<=REGEXP_COUNT(S,b)/3 );
````
Obviously, you can always write convoluted code for fun, but sometimes you come against actual queries:
- which update data
- reading from a dozen tables, going through many intermediary result set
- using analytic functions (window functions)

You may even see your code is calling several queries in the same transaction: 
````sql
INSERT (..) 
UPDATE (..) 
INSERT (..) 
DELETE (..) 
````

So, can you test it properly ? Can you write a regression test before implementing your bugfix ? 
You may answer:
- such queries are batch processing, and batch processing should be banned 
- these queries are at the wrong place, such data processing should not be done in the database
- we have a full end-to-end regression test which take place in the week-end, so I won't write any test on this level

But if you can't afford these answers (at least, for the next couple of weeks), you may be interested
with more attempts in doing automated testing on the database. 

# Interlude: globals considered harmful 
We need to understand why is database testing that hard. 
Let's take a step back and let me tell you an anecdote on global variables.

Global variables can save you a lot of time AND cause you headache.
We've been warned about it [since 1973](https://dl.acm.org/doi/10.1145/953353.953355).

Environment variables are very global variables, on OS level.
Last month, my pairing partner and me to spend a whole afternoon to make a test pass.
The test was running successfully since the beginning... by itself, in the IDE. 
Not in the whole test suite.

We ended up printing all environment variables in the console, every time a test was run, in the whole test suite.
We've already reached a pretty bad state. After we add a single line, and the problem was solved, we 
were in utter despair.
``` javascript
delete process.env.MY_VARIABLE;
```

You know the way out of it:
- on application start, read the configuration from environment variable
- injecting this configuration using arguments 
  
You also know such refactoring can be time-consuming, so make you best guess before to proceed.

I did this for educational purpose on my spare time, and went as far as:
- removing all environment variable from the process memory
- asking the linter to prevent any non-intended use in the future

If you read javascript, the following code extract is for you to peek.

Configuration is injected on application start
`replication_job.js`
```` javascript
const extractConfigurationFromEnvironment = require ('./src/extract-configuration-from-environment');
const configuration = extractConfigurationFromEnvironment();

new CronJob(configuration.SCHEDULE, async function() {
  await steps.fullReplicationAndEnrichment(configuration);
  });
````

It is extracted there
`extract-configuration-from-environment.js`
```` javascript
/* eslint no-process-env: "off" */

const extractConfigurationFromEnvironment = function() {
  loadEnvironmentVariableFromFileIfNotOnPaas();
  const extractedConfiguration = extractConfigurationFromEnvironmentVariable();
  removeConfigurationFromEnvironmentVariable();
  return extractedConfiguration;
};

const loadEnvironmentVariableFromFileIfNotOnPaas = function() {
  if (process.env.NODE_ENV && (process.env.NODE_ENV === 'production' || process.env.NODE_ENV === 'test')) {
    return;
  } else {
    require('dotenv').config();
  }
};

const extractConfigurationFromEnvironmentVariable = function() {
  return {
    PG_CLIENT_VERSION : process.env.PG_CLIENT_VERSION || 12,
    (..)
  };
};

const removeConfigurationFromEnvironmentVariable = function() {
  delete process.env.PG_CLIENT_VERSION;
  (..)
};

module.exports = extractConfigurationFromEnvironment;
````

Any other use of environment variable will trigger a linter error

`.eslintrc.yaml`
```` yaml
rules:
  no-process-env:
  - "error"
````

The original refactoring is available in [this PR](https://github.com/1024pix/pix-db-replication/pull/47).

# General-purpose language blessings

What does this globals story tell us ?
In a high-level general purpose program, you've got considerable power to prevent your program
accessing values from anywhere, making it much more deterministic, therefore easy to test.

## Memory management
Let's start with the memory. In most languages, the variable scope is either local or pass-by argument.
If you do the following, your component behaviour is easy to guess:
- reading from arguments 
- computing using local variables
- returning a result

This does magic on testing. Of course, as soon a global variable come is, thing get worse.
Now consider you have to test some C code.
It is using malloc by hundreds and can read and write anywhere in its memory space.
Your component is much more prone to bugs, and writing a test to find them much more difficult.
You have to check it didn't change *in the memory space* any data that were not intended to change.
You may object it can't write out of *its own* memory space without immediate [punishment by OS](https://en.wikipedia.org/wiki/Segmentation_fault).
True, but you don't want it to happen in production, right ?

To sum it up, enforcing rules on memory management is good for testing.  

## Side effects 

Memory is but one of the many side effects a program can cause.
Filesystem, network, logging, time are such other ways a program can read and or write data. 

If side effects are implemented by a few components, your application is much more manageable to write and to test.
Interface components, MVC models, many solutions have been designed to achieve such isolation.
In the last decade, I can single out  [hexagonal architecture]([https://blog.octo.com/en/hexagonal-architecture-three-principles-and-an-implementation-example/).

# Back to database 
So, this detour was intended to make you think about:
- database as memory
- database tables as globals variables.

### Database as memory

The fact that database is memory can look frivolous, since today all system are [Von-Neumann](https://en.wikipedia.org/wiki/Von_Neumann_architecture) implementation,
so programs and data are no longer persisted in separate media such as punch cards. 

If we look at it in greater details, database implements two distinct features (including non-relational databases):
- store data on the filesystem: you don't have to bother managing many files (in a way that use little space while preserving access time)
- process data: an INNER JOIN followed by GROUP BY can be implemented using successive arrays (in a way that use little space while preserving access time)

Now if we focus on storing data, it boils down to the mere fact that the filesystem is a memory. Strictly speaking, memory is primary storage, while a filesystem on top
of hard drive or tape is secondary storage. If we keep with the anthropomorphic metaphor, and find a system with infinite memory, no access time, infinite computing
power and no failure, database may not exist at all.     
> The issue is how well the RelationalModel supports my efforts as application programmer to support the lie that I have an infinite amount of transactional memory.
[C2 wiki](https://wiki.c2.com/?ObjectRelationalImpedanceMismatchDoesNotExist)

### Tables as global variables

In SQL, you can read/write any table, like a global variable.
You can restrict access using a privilege system, but its is set once, not before each query, so they are very similar.

As we saw earlier, this is usually not a problem, and is indeed a very welcome fact.
We're happy to access many tables at once, going as far as using database link to access a remote database and creating datawarehouses.

This come with downsides, as with globals. 
I found the worst is regression on data modification: you alter data you didn't mean to.
Fixing a bug in a convoluted SQL query is like playing Mikado:
- it's pretty easy to pick out this stick...
- ... but it's pretty hard if you also consider not moving its neighbors.

It happens in SQL queries in applications which:
  - mostly use a general-purpose language, but had to move the processing (not storage) in SQL queries because of a performance problem 
  - that been designed from the beginning with SQL holding a large slice of the pie  

It also happens in applications which relies on procedural code stored in the database, having native access to tables.
- [Oracle PL/SQL](https://framagit.org/parcoursup/algorithmes-de-parcoursup/-/blob/master/plsql/ordreappel/calculeOrdreAppel.sql) 
- [Postgres Pg/PL SQL](https://github.com/yahoojapan/big3store/blob/c4142395695a836fec60b7c202f9f12d46a8ed01/src/plpgsql/encode.plpgsql)

It's not easy to point out the original rationale in favor of procedural database code, so I'm guessing:
- performance: code executing in the same memory space as the database engine has much less latency,
- skills: if you've been an Ada developer and know SQL basics, the ramp-up is quicker in such languages than if you chose Java. 

# Some ways to achieve database testing

## The goal
Writing tests that prevent regression in data involve checking *nothing has changed* in:
- a table
- a table, but some rows

You know how to check that: compare the value before and after exercising the SUT. 

What I propose here is merely :
- more convenient, as you don't have to write specific code for each table,
- more efficient, as storing each data twice to compare them can be costly.

I won't use any library, only native solutions. Code sample is PostgreSQL, but can be ported.

## What to check ?
Our first impulse would be to check the data after exercising SUT.
To make sure we wouldn't miss anything this way, I take Vladimir Khorikov's checklist from [his unit testing book](https://www.manning.com/books/unit-testing).

There are 3 ways to assert an expected result of a component:
* check the state of an object he acted upon
* check the behaviour of the component: which components did he call, with which arguments ?
* check the value returned by the component. 

## Assert behaviour
It may look like checking the behaviour isn't an option in SQL.
As Scott Wlaschin [pointed out]](https://www.youtube.com/watch?v=0fpDlAEQio4), SQL is high-level functionbal language.
You tell him what you want (find and update this row), he'll decide on how to do this (open datafile, etc..).
Actually, there's several ways

### Table level: privilege
We can leverage database built-in privileges if you work at table-level.
You will be able to check if a table has been read or wrote.
You won't be able to check that `some` rows has been altered, but not others.

If you expect a table
- not to be accessed at all, wipe out all privilege
- to be read, not written: grant read privilege only
- to be written: grant write privilege only

When you exercise the SUT, if a privilege error pop up the logs, you know he's been doing something he shouldn't have done.
https://github.com/GradedJestRisk/db-training/blob/master/RDBMS/PostgreSQL/detect-data-change/using-privileges.sql

### Component-level: test double
If your SQL is wrapped in a procedural object, you can implement a test double library
Actual calls will be logged in a table and checked against your expectations.

[Implementation here](https://github.com/GradedJestRisk/db-training/blob/master/RDBMS/PostgreSQL/detect-data-change/using-test-double.sql)


## Assert value returned 
This option is available in PostgreSQL.
Just add a `RETURNING` clause at the end of you INSERT, UPDATE or DELETE query, and the affected queries 
will be returned in a dataset.

I wouldn't consider this option as you're changing production code in order to test it.
Furthermore, huge amount of data may be involved, filling up network and memory.  

## Assert state

### Database level: dump
If you want to make sure *nothing* has changed in the whole database, you can leverage OS `diff` skills.
Just make data available through dumps, and you can type in:  
`diff --context before.sql after.sql`

You don't need to export your whole database: segregate the candidate table into a schema and dump that schema.
[Implementation here](https://github.com/GradedJestRisk/db-training/blob/master/RDBMS/PostgreSQL/detect-data-change/using-dump.sql)

### Row level: digest

#### Simple solution
Most of the time, you don't need any of the previous solutions, as reading the query itself is enough.
You want to check if `some` rows has been altered.

We don't care about the actual value before/after the change, but rather that it didn't change.
Sound familiar ? Yes, we need some digest. Take a snapshot of each row, in a digest, before and after exercising SUT.

[Implementation here](https://github.com/GradedJestRisk/db-training/blob/master/RDBMS/PostgreSQL/detect-data-change/using-digest.sql)

You'll trade computation power (to extract digest) with disk space. This digest is 64 characters, so if you've many columns
with much data (say, VARCHAR(200) or JSBONB ), it will do the trick.

#### Full-blown solution

When you're talking about digest and tracking data change, chances are a versioning tool is on his way.
Someone thought about it and created a version-control for database data, rather than database structure.
[Implementation here](https://github.com/mmkal/plv8-git)

# So what ?

Many things can be done in many ways, and as many assertions libraries can be written as in 
general-purpose languages. I had to tell it hasn't been done use on production code.
Maybe because it overlaps 2 programming cultures that haven't met yet ? 
