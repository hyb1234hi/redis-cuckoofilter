redis-cuckoofilter &middot; ![CircleCI Status](https://circleci.com/gh/kristoff-it/redis-cuckoofilter.svg?style=shield&circle-token=:circle-token)
==================
Hashing-function agnostic Cuckoo filters for Redis



What's a Cuckoo Filter?
-----------------------
Cuckoo filters are a probabilistic data structure that allows you to test for 
membership of an element in a set without having to hold the whole set in 
memory.

This is done at the cost of having a probability of getting a false positive 
response, which, in other words, means that they can only answer "Definitely no" 
or "Probably yes". The false positive probability is roughly inversely related 
to how much memory you are willing to allocate to the filter.

The most iconic data structure used for this kind of task are Bloom filters 
but Cuckoo filters boast both better practical performance and efficiency, and, 
more importantly, the ability of **deleting elements from the filter**. 

Bloom filters only support insertion of new items.
Some extensions of Bloom filters have the ability of deleting items but they 
achieve so at the expense of precision or memory usage, resulting in a far worse 
tradeoff compared to what Cuckoo filters offer.

For more information consult
[the project's wiki](https://github.com/kristoff-it/redis-cuckoofilter/wiki/1.-Cuckoo-VS-Bloom).

What Makes This Redis Module Interesting
----------------------------------------
Cuckoo filters offer a very interesting division of labour between server and 
clients.

Since Cuckoo filters rely on a single hashing of the original item you want to 
insert, it is possible to off-load that part of the computation to the client. 
In practical terms it means that instead of sending the whole item to Redis, the
clients send `{hash, fingerprint}` of the original item.

### What are the advantages of doing so?
	
- You need to push trough the cable a constant amount of data per item instead 
  of N bytes *(Redis is a remote service afterall, you're going through a UNIX 
  socket at the very least)*.
- To perform well, Cuckoo filters rely on a good choice of fingerprint for each 
  item and it should not be left to the library.
- **The hash function can be decided by you, meaning that this module is 
  hashing-function agnostic**.

The last point is the most important one. 
It allows you to be more flexible in case you need to reason about item hashes 
across different clients potentially written in different languages. 

Additionally, different hashing function families specialize on different use 
cases that might interest you or not. For example some work best for small data 
(< 7 bytes), some the opposite. Some focus more on performance at the expense of 
more collisions, while some others behave better than the rest on peculiar 
platforms.

[This blogpost](http://aras-p.info/blog/2016/08/09/More-Hash-Function-Tests/) 
shows a few benchmarks of different hashing function families.

Considering all of that, the choice of `{hashing func, fingerprinting func}` has 
to be up to you.

*For the internal partial hashing that has to happen when reallocating a 
fingerprint server-side, this implementation uses FNV1a which is robust and fast 
for 1 byte inputs (the size of a fingerprint).*

*Thanks to how Cuckoo filters work, that choice is completely transparent to the 
clients.*

If you know how Cuckoo filters work and are interested in knowing the implementation
details of this module, please consult
[the project's wiki](https://github.com/kristoff-it/redis-cuckoofilter/wiki/2.-Implementation-details).


Installation 
------------

1. Download a precompiled binary from the 
   [Release section](https://github.com/kristoff-it/redis-cuckoofilter/releases/) 
   of this repo or compile with `make all` (linux and osx supported)

2. Put the `redis-cuckoofilter.so` module in a folder readable by your Redis 
   installation

3. To try out the module you can send 
   `MODULE LOAD /path/to/redis-cuckoofilter.so` using redis-cli or a client of 
   your choice

4. Once you save on disk a key containing a Cuckoo filter you will need to add 
   `loadmodule /path/to/redis-cuckoofilter.so` to your `redis.conf`, otherwise 
   Redis will not load complaining that it doesn't know how to read some data 
   from the `.rdb` file.


Usage Example
-------------

```
redis-cli> CF.INIT test 64K
(integer) 65536
 
redis-cli> CF.ADD test 5366164415461427448 97
OK

redis-cli> CF.CHECK test 5366164415461427448 97
(integer) 1

redis-cli> CF.REM test 5366164415461427448 97
OK 

redis-cli> CF.CHECK test 5366164415461427448 97
(integer) 0
```

You can find the complete command list on
[the project's wiki](https://github.com/kristoff-it/redis-cuckoofilter/wiki/4.-Module-Commands-(API)).


Choosing the right settings
---------------------------
Consult [the project's wiki](https://github.com/kristoff-it/redis-cuckoofilter/wiki/3.-SIZE,-FPSIZE-and-Error-Rate).


Testing 
-------

You can find how to run tests on
[the project's wiki](https://github.com/kristoff-it/redis-cuckoofilter/wiki/5.-Testing).

Planned Features
----------------

- Cuckoo filters for multisets: currently you can add a maximum of 
  `2 * bucketsize` copies of the same element before getting a `too full` error. 
  Making a filter that adds a counter for each bucketslot would create a filter 
  specifically designed for handling multisets. 

- Dynamic Cuckoo filters: resize instead of failing with `too full`


License
-------

MIT License

Copyright (c) 2017 Loris Cro

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
