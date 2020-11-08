# TL,DR
I'm using mostly open-source code for a year, and that's quite a revolution regarding individual learning opportunities.
I want to share this experience, on a traditionally experts-hands-only domain: performance.
This will take place using an actual SPA-API-DB stack code, actually used by 2 millions users (time of this writing): 
[Pix](https://github.com/1024pix/pix). Actually, it will take place on your laptop.    

# Meta 
First things first, let's write about writing.
If you're only interested in performance stuff, skip this part and jump to [Actual stuff] 

## Why am I to write anyway ?
> But do not think that just experienced professionals should write blogs. (...) 
Sometimes we may think that we are not good enough or do not have much to say.
We may think that we don't have an original idea and no one will read our blog anyway. 
First of all, we should treat our blog as a record of our own learning and progression  - a history 
of our thought, ideas and views of the world over our careers. We should not worry too much
about what other people will think about it. We should first write it for ourselves.
Sandro Mancuso, The Software Craftsman

Since a couple of weeks, I feel an urge to write. Write things down, as ideas are about to coagulate in my head, and
I need an extra push to make this happen. You can't always hold so much ideas in you mind, but on paper you can manage many more

So, the kind of ideas that have been around for while, but you didn't even notice them. Till someday the 
pieces fits together, and you find a solution for a problem you was not consciously aware of.

## Are you kidding ? An article in a markdown file ! 
Sure ! I even use trunk based development. I can wonder if I shall use Medium and learn more CSS, or install a CMS on a PaaS,
or sollicitate my current company blog, or my current client blog, going through review workflow, actual publishing. 

But while doing so may be interesting, you won't be reading these lines before a month
I actually need quick feedback, so I need to reduce [waste](https://en.wikipedia.org/wiki/Lean_manufacturing).
As for now, the only feature I may consider install a [markdown linter](https://github.com/GradedJestRisk/js-training/blob/master/node/code-quality/lint/.eslintrc.js#L7) for proofreading.

So let's `watch -n 60 "git add --all && git commit --amend --no-edit && git push --force-with-lease"` and go on !
 
## Why did you choose performance tests as a subject ?
I spent my professional career (12 years) mostly in backends, writing (Ada-like) structured code ([PL/SQL](https://en.wikipedia.org/wiki/PL/SQL) in Oracle database actually).
I also spend time in backend suchs as bakeries and high furnace, but that's another story. 
You may think I ended up with much DBA skills, but I was prevented such opportunity.
In the places I went, the less you know and came to know, the better it was for management.
 
2 years ago, I decided to part company with the company. 
I'm actually using mostly open-source code, and that's quite a revolution in several concerns:
- at work, your playground isn't limited anymore to proprietary technologies you practiced (eg. PowerBuilder)
- at work, you can try out ideas without using borderline strategies (eg. using production environment without DBAs knowing it)    
- you can practice at home, on your personal laptop (try this with Oracle Express !)

I personally felt some kind of empowerment having taking place.
 
I want to share this experience, especially with people: 
- who never worked with proprietary code, and are unaware of their actual good fortune,
- who always worked with proprietary code, and are unaware of what life is about (this may be a joke).

To come back to the point, running performance test on my laptop was as the time something I would never have considered.
This would have involved database, application servers, licence servers, VMs and all that operation stuff.
So, after I spend a day in the weekend to try this out, and it worked, this was the ultimate proof for me.
    
# Actual stuff

## Some (simplified) context for those requiring it

I'm working as a contractor for pix.fr, a public-sector company offering web-based skills assessment.
Anyone can use this service, but pupils/students are expected to use it as certain law has been passed recently.
(As for now, I will refer to them both as students, even thought they can belong to any level of educational system).

This has gone so far as considering that some students should get certified skills.
This set a strong incentive for cheating, especially using leaked assessment materials.  
To avoid this, each teacher has to provide a list of allowed students (basically, a class list).

As each student log in the application using a custom [identity provider](https://fr.wikipedia.org/wiki/Gestionnaire_d%27acc%C3%A8s_aux_ressources) 
the application could check if the person requesting access was legitimate to do so.  

This list wasn't live-fed through API calls or any messaging system.
It was a XML file based extraction, manually imported by the teacher.
Let's call it schooling-registration file.

## Passing by: identification, authentication, authorization
To enforce security (that is, avoid cheating), we need to identify students, even thought they log using 
[another system](https://fr.wikipedia.org/wiki/Gestionnaire_d%27acc%C3%A8s_aux_ressources) providing authentication. 

> Authentication is the act of proving an identity of a computer system user. 
> In contrast with identification, the act of indicating a person or thing's identity, authentication is the process of verifying that identity. 
From [WP](https://en.wikipedia.org/wiki/Authentication)

## The problem at hand 

Several weeks ago, we come against repeated containers crashes after OOM. 
While not able to pinpoint the root cause, we noticed that before each crash, a schooling-registration file import had happened.
    
Upon further analysis:
- the files involved were pretty bigger than usual
- the file import recently underwent a major change.  

Much as been written about batch processing VS small unit, as either One-Piece flow in TPS or metaphorically in IT data processing.
A colleague of mine went so far as saying that telecom companies would save them a lot of hard work switching to single-call immediate billing to customer.
Anyway, such solutions as the following could not be build one day:
- create a complete API flow between our system and the external one
- keep the file integration, but in dedicated containers that wouldn't propagate failure to standard queries. 
  
So we set up to find the bug and fix it.
  
At first glance, the memory footprint you get using xmldom library is quite impressive.
For this bare-to-earth XML (here in JS for convenience - 124 Bytes), you end up [using 22 KBytes](https://github.com/GradedJestRisk/js-training/blob/master/node/parsing/XML/XML-to-JSON/parse-memory-footprint.js). 
```javascript
{ greatMother:
   { name: 'Susie',
     mother:
      { name: 'Barbara', daughter: { name: 'Alice', text: 'Hey!' } } } }
```

This pull request aims to fix the problem by switching from loading the entire file into memory to parsing using streams.      
https://github.com/1024pix/pix/pull/2061
 

## A gradual approach 
  
## Step 1 : A bootstrap (load-testing with Artillery)

## Step 2 : All things being equal (enforcing quota with Docker)

## Step 3 : The end (plugging things together and fixing bugs)
https://github.com/1024pix/pix/pull/2109

# So what ? (conclusion)   
