# Antipatterns in data access, part 1 - memory joins

*I wanted to write this for a long, long time ...*

So, having to work on various projects for the last decade or so - I've noticed that a lot of them are having troubles and issues with a part that communicates with their database.

Being such a delicate part of the system - very often it is completely impossible to fix and refactor. Or at least very difficult and risky. So, in most cases I've seen, developers just don't dare even to touch it - and just leave it be as is.

Now, there are certainly always things that can be done, nevertheless, it is always best strategy just to avoid those anti patterns from very start.

So what are they?

Well, literature and blogosphere around this topic is scarce and a bit indeterminate. There aren't many articles discussing this important issue details.

Found this one though: https://www.infoq.com/articles/Anti-Patterns-Alois-Reitbauer/

But it is still bit general in my opinion and it confines itself to performance. While those antipatterns do have dramatic impact on performances - they are not strictly speaking just performance antipatterns. They are structural in nature also in my opinion.

I'll try to describe in details all of them that I can remember from working with broken applications for decades. Ones that keep recurring and recurring at least.

So, this will be series of articles as I try to describe each and every one, as I remember them.

If you don't agree with me, and think that some of those approaches aren't anti patterns - please let me know politely. I'm always in for different perspectives.

## Antipattern 1: Joining datasets in database client.

Joining different datasets into one has always been one of the core database functionalities. And as such, it is excellently supported by ORM libraries. Nevertheless, some developers would opt out from using that joining functionality, for different reasons, and build solution that resembles something like this:

- **Query first data set** and fetch data to database client (your program).

- **Query second data set** and fetch data to database client (your program).

- **Construct algorithm** in your program that will join those two data sets into one (as required) - inside your program memory.

Now, reasons why developers adapt this approach may differ. For example:

- In some cases ORM tool will lack expressiveness for queries of that complexity (like for example Django ORM) and developers under impression that SQL must be avoided by any cost.

- Developers have reached to conclusion, for some reason - that join (or merge) operation is part of the domain responsibility. When working under DDD methodology for example.

- They simply don't know better way. Lack of proper mentoring and unrealistic confidence.

So, whatever the reason may be, what is the problem with this approach and why is this antipattern? Couple of reasons in my opinion:

- **It is slow.** It is very slow. I'll explain shorty...

- **It is not very scalable.** More data you get into your system slower it gets until you reach critical point when it becomes totally unusable. Then you have no choice but to outright rewrite large portions of your application or you will lose important customers.

- **It is dangerous.** It may eat up your entire memory.

- **It adds to maintainability and complexity.** You have to write complicated algorithm manually. Compare that to one database instruction or directive.

So what makes it so slow?

First, there is an issue of bandwidth, you need to transfer a considerable amount of data over network, and then you have to iterate again, at least three times (assuming that you narrowed time complexity of your algorithm right).

As someone brilliantly once described it visually on Twitter - you think of it like this:

**Data has a mass** (no, not really, but we may look at it like that - considering a fact data size and time for data transfer over network are proportional). And large data set can be viewed as a mountain.

In this case we have two actually mountains. And what you really are doing with this approach is moving two mountains to your toolbox to be able to work on them. That, essentially, is what it is. And how can anyone can think that's performant or scalable? It is not.

And not only that you are literally moving mountains to achieve desired results. When you move your toolbox to the mountains (that is, use SQL that works directly on database) - database algorithm will always outperform whatever you may written in your program for at least two major reasons: 1) works directly on database obviously 2) database optimizer will in most cases select better algorithm than you do, because it is selected based on available data of your table statistics. That means if your data in production is much larger then in development database, selected algorithm may be different and much more appropriate.

Performance gains that I've got by rewriting this antipattern to SQL queries are anywhere from 50 to 200 times faster (without even touching indexes) on moderate datasets (tables around 500K records approximately).

And all of that by 2 to 3 times less code which simplifies future maintenance efforts.

I also must point out two important things:

- Rewriting this antipattern was always inevitable in all cases (loss of customers due the unusability)

- Rewriting this antipattern was always extremely tedious and hard and often demanded multiple testing and fixing cycles.

- Code that needs to be changed is often scattered across the system and often obfuscated and masked as something else - which makes rewrite and refactor so much difficult and risky. Especially in dynamic languages.

So, in order to avoid such undesirable situations this antipattern should be avoided from very start. Simply write your query in your database language whatever it is. Database language is your data API and it should return all that your program needs without any further data operations (joins, merge, union, etc).

If you are using ORM, fine, first explore that ORM supports that type of operation (join, merge, union, whatever it is) - that you require and that doesn't force you to drag data to program memory.

If it doesn't then you must write proper database query that does return everything you need, usually SQL (there are other like CQM, Cypher, etc). If mixing of languages makes you uncomfortable and you considered as code smell - separate it to another file. Well, everything is better then dragging a mountain of data to your program.

And do write some tests around that query, it will make further optimizations much easier.

So what do you think?

Do you consider this approach to be antipattern at all? Have you ever encounter this in production? I'd love to hear some thoughts.

Next week I'll discuss another nasty data access antipattern which is very common unfortunately. Markus Winand refers it as "nested loops antipattern", which is pretty accurate. Christian Baune refers it as "machine gun data access". Anyhow, it suffer same symptoms and it is often coupled with this antipattern I just described.

And so, until the next time...
