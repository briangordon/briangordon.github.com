---
layout: post
title: MagicalTux's EVE Online pathfinder
tags: programming
---

The personal blog of Mark Karpeles, aka MagicalTux, CEO of the recently-bankrupt Bitcoin exchange Mt. Gox, [received attention](https://news.ycombinator.com/item?id=7295932) on Hacker News recently. Given that Gox's spectacular $473 million downfall was supposedly caused by a bug in Karpeles's custom implementation of the Bitcoin protocol, people were understandably interested in checking the quality of his public source code.

What we saw was not reassuring. In [one post](https://web.archive.org/web/20140226001727/http://blog.magicaltux.net/2010/06/27/php-can-do-anything-what-about-some-ssh/), Karpeles describes a custom implementation of SSH2 which he wrote for production use at his web hosting company.

> With PHP I could write a fully working SSH server in only 3 days. ... My goal when writing this was to provide a replacement for the FTP protocol for the customers of my hosting service.

> As I was missing some functions to properly handle the initial Diffie-Hellman key exchange (and later to implement publickey authentication) I had to re-implement those in PHP.

This is horrifying to see from the guy who wrote the server which once handled [70 percent of all Bitcoin trades](http://www.wired.com/wiredenterprise/2013/11/mtgox/all/). If I had seen this post before Mt. Gox's failure, I would never have deposited my Bitcoin with them. The SSH server source code has since been taken down, but one HN user [remembers it](https://news.ycombinator.com/item?id=7335178):

> The code for the sshd does not seem to be there anymore, but from memory: it did not check if the number sent by Bob was 0, 1, or any any other groups that would make it easy solve the discrete logarithm problem. I don't think it bothered to check the primes either. [1] I think there was also something wrong with the signature checking (padding not checked maybe?).

> Altogether it seemed like you could easily MITM connections made to the server, but I don't think I ever tried. It was a perfect example--to me at least--of why you should not spend a trivial amount of time reading about crypto on Wikipedia and then writing crypto code.

I absolutely agree. The main lesson that you should take away from [the crypto class](https://class.coursera.org/crypto-preview/lecture) is that you should be afraid of rolling your own crypto code. There are a thousand ways to screw up, and it only takes one mistake for your cryptosystem to fall apart. The recent [Apple](https://www.imperialviolet.org/2014/02/22/applebug.html) and [GnuTLS](http://arstechnica.com/security/2014/03/critical-crypto-bug-leaves-linux-hundreds-of-apps-open-to-eavesdropping/) vulns show that even the serious players get this wrong.

So best practices were apparently not followed at Mt. Gox. In fact, later [it was alleged](http://www.wired.com/wiredenterprise/2014/03/bitcoin-exchange/) that developers at Mt. Gox would push changes directly to production, and didn't even use version control for the site's source code.

[Another of his blog posts](https://web.archive.org/web/20101125085812/http://blog.magicaltux.net/2008/11/30/eve-online-pathfinder/) is about a tool which he wrote - in PHP of course - to compute routes between star systems in the MMORPG [EVE Online](https://en.wikipedia.org/wiki/Eve_Online). In EVE, solar systems are connected to each other with portals called Stargates. The result is [a big graph](http://evemaps.dotlan.net/). Savvy players will try to take the shortest possible path to get from point A to point B by using a shortest-path algorithm to automate navigation.

![Systems map in EVE Online](/images/eve-systems.png)

I thought I'd hop on [the bandwagon](https://news.ycombinator.com/item?id=7332391) by criticizing his EVE pathfinder. Although I could nitpick on matters of style, I'd rather focus on the core- that is, the [all-pairs shortest path solver](https://en.wikipedia.org/wiki/Shortest_path_problem#All-pairs_shortest_paths) which is the basis of his algorithm. The general idea is to generate an index which contains, for each system, the next hop to take in order to reach any given system. This uses O(n^2) space but allows efficient pathfinding between any two arbitrary systems in the universe. All of this is perfectly fine so far. The problem is how he goes about constructing this index.

Based on my reading of his code, this appears to be his algorithm:

1. Inform each system how to reach its adjacent systems.
2. From each system `s`, collect the best known paths to systems 1 hop away and advertise them to `s`'s adjacent systems.
3. From each system `s`, collect the best known paths to systems 2 hops away and advertise them to `s`'s adjacent systems.
4. From each system `s`, collect the best known paths to systems 3 hops away and advertise them to `s`'s adjacent systems.
5. From each system `s`, collect the best known paths to systems 4 hops away and advertise them to `s`'s adjacent systems.

This continues for 256 steps, by which point MagicalTux hopes all systems have been informed of the shortest path to all other systems. Judging by a cusory search of the EVE forums, [this appears to be a valid assumption](http://oldforums.eveonline.com/?a=topic&threadID=1363890). 

This algorithm is essentially the [Bellman-Ford algorithm](https://en.wikipedia.org/wiki/Bellman%E2%80%93Ford_algorithm) run once for each system in the universe. Although this solution is asymptotically optimal, in real life it will perform poorly compared to a real all-pairs shortest path algorithm. I thought it would be interesting to see how badly I could beat his code's performance. MagicalTux claims that it takes **about 3 hours** to construct the index using his PHP code. I'm going to see how much faster I can do it, by doing these things differently:

* Use C++ instead of PHP.
* Don't use file I/O as a working data structure.
* Use [Floyd-Warshall](https://en.wikipedia.org/wiki/Floyd%E2%80%93Warshall_algorithm), optimized as best I can, instead of his algorithm.
* Use OpenMP to parallelize the algorithm.

I downloaded the jump connection data helpfully provided by MagicalTux - the [official data](http://community.eveonline.com/community/fansites/toolkit/) comes in an unusable binary MSSQL format - and put it in [a CSV file](https://gist.github.com/briangordon/8fa812fecccad11e1f17) for easy parsing.

My complete EVE pathfinder implementation can be found [here](https://gist.github.com/briangordon/9330167). It doesn't require any extra space (asymptotically) to construct the index, which is nice. This is the core of the code where almost 100% of the running time is spent:

~~~ cpp
// Floyd's algorithm
for(uint32_t k=0; k<NUM_SYSTEMS; k++) {
    #pragma omp parallel for shared(k, cost, next) schedule(dynamic)
    for(uint32_t i=0; i<NUM_SYSTEMS; i++) {
        if(i == k) {
            continue;
        }
        for(uint32_t j=0; j<NUM_SYSTEMS; j++) {
            uint32_t prev = cost[(NUM_SYSTEMS * i) + j];
            uint32_t candidate = cost[(NUM_SYSTEMS * i) + k] + cost[(NUM_SYSTEMS * k) + j];
            if(candidate < prev) {
                cost[(NUM_SYSTEMS * i) + j] = candidate;
                next[(NUM_SYSTEMS * i) + j] = next[(NUM_SYSTEMS * i) + k];
            }
        }
    }
}
~~~

I picked two random system IDs (30000029 and 30000050) for a pathfinding demo. My test program constructed the all-pairs index and then used it to find the shortest path between the two systems. You can see the output of the test below:

    brian@mint ~/eve $ ./eve 
    Indexing... done. Elapsed time 38.99 seconds.
    Calculating the quickest route from 30000029 to 30000050... done.
    Built a 14 hop route in 0.002 ms.

Note that, indeed, the shortest route between those two systems [according to DOTLAN](http://evemaps.dotlan.net/route/Lachailes:Fera) is 14 hops! And the index was built in less than 40 seconds- about 0.4% of the time that it took MagicalTux's PHP version.
