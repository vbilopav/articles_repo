# Antipatterns in data access, part 2 - nested selects

In first part of antipatterns in data access series of articles - [I've discussed memory joins antipattern.]()

Nested selects anti pattern is very much similar, because, it is yet another attempt (by different means) - to perform database joins on client.

> Database joins need to be executed in the database.

## **Ramifications are pretty much the same:**

- Badly under-performing system or suboptimal performance to put it mildly.

- Compromised vertical scalability - more data in system worse it progressively underperforms - until it reaches breaking point (unacceptable response time for the user).

- Fixing requires very risky, tedious and difficult rewrites of large portions of your application (often within very restricted time frame because users urgency which leads to high levels of stress that nobody needs or wants).

## **So what is it?

Well, it is attempt to join the data on client, from two different queries which can be best described with following pseudo code:

```
for each row from outer query:
   
    for each row from inner query:
  
        ## join results to new data structure from query 1 row and query 2 row

    end loop

end loop
```

It is also known as **N+1 query problem**, because it generates N+1 queries - one for initial outer query and N queries for N results of initial outer query - equals N+1 queries.

## So why is it so slow?

It's because you need to execute N+1 query where N could be hundred or thousands of queries, or sometimes even more. And every query will return data after it is executed on database plus your network response time (or latency) multiplied by number of queries, plus bandwidth and database execution time.

**Network response time (or latency) is main factor here.** Unlike previous anti pattern (memory joins) where bandwith was bottleneck - this is all about latency. Minimal time to execute this type operation would be Latency * (N + 1), where N is number of queries. And you still have bandwidth and database execution to deal with, which is even minor factor here.

So it is slow indeed, and no matter how you index your database (it is usually over indexed anyhow), it is still going to be slow. And not only that, with large amount of unnecessary code behind it, that it will sooner or later have to be urgently rewritten.

Nevertheless, in my experience, it is very, very common approach, whatever the reason. Usually junior developers without mentoring or training in database development tasked in building business application. And sometimes even experienced devs used in working in certain ways.

Entire enterprise systems built entirely by using this antipattern. Dozens and dozens of searchable data grids, dozens of reports - all built with this anti pattern. Very slow, on very edge of usability and very hard or impossible to scale, and even harder to rewrite properly.

At beginning of my career I was mentored to avoid this anti pattern by any means. And at that time, it seemed like common sense and general knowledge. Today it seems like, at least to me it is generally accepted approach, even in very serious companies. Hardware got lot stronger, so it's not visible so much... at first in the beginning when data is small.

In my opinion, it is still best to avoid this antipattern completely from very beginning.

So, what do you think? Do you agree with me? Have you noticed this anti pattern much? Do you think it is anti pattern at all? Let me know what you think.

Next episode, I'll be discussing one also very common data access antipattern - overselecting.
