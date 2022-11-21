[https://www.reddit.com/r/rust/comments/sr02aj/what_makes_ripgrep_so_fast/](https://www.reddit.com/r/rust/comments/sr02aj/what_makes_ripgrep_so_fast/)

Well that code appears to be doing a byte-at-a-time loop, which is exactly what is slow compared to SIMD. Try modifying your code to split lines using memchr.
But yes, you're right, there is another trick here. ripgrep does not typically iterate over every line in a file. It just searches groupes of lines as one big chunk. If a match is found, it is only then that the corresponding line is found. This optimization rests on the assumption that matches tend to be rare. It is overall slower (but only a little bit) if there is actually a match on every line.
So yes, it seems like your understanding is correct. But as I mentioned, that is not the only difference from your code. Your code (likely) doesn't use SIMD.

The [bstr](https://crates.io/crates/bstr) crate includes a `BufRead` extension trait which adds [functions for efficient line iteration](https://docs.rs/bstr/latest/bstr/io/trait.BufReadExt.html#method.for_byte_line), lending out slices of the internal buffer wherever possible, and using a growable auxiliary buffer only for trailing lines and those which don't fit.

Before writing those I made [linereader](https://crates.io/crates/linereader), which only supports a fixed buffer - this does have the disadvantage you mention of potentially having partial lines to deal with, but that also comes with the benefit of predictable memory use, so it depends on what your requirements are.

#################

arm has assembly implementations, musil has optimized version in c

If you have a ‘really rare byte that you feed to that memchar routine, it will stay in memchar for the vast majority of that routing’, if you have a popular byte.

1) Smart filtering (ignores .git_ignore, global ignore files, ripgrepignore flies, those take time to process into globs.)

2) Parallelism - grep isn’t very parallel

3) Regex Engine itself

- Literal optimizations:

When gnu grep implements boyer-moore, the main thing you’re suppossed to remember is that it will skip characters, it will look at a character and it will consult a ‘mismatch table’, if that char doesn’t appear in your search string somewhere, the algorithm knows to skip X chars and move on to the next piece.

Mike hartell’s post in the BSD mailing list, it popularized this thing, hey we don’t actually look at every byte. Turns out on modern hardware (this may not have been true back when Mike Cartel had written that mailing list post) but nowadays on modern hardware that isn’t the thing that makes boyermoore fast. It’s actually the skip-loop, it takes the last byte of the pattern and feeds it into memchar. Memchar is provided by memc, on most platforms ‘take a single byte and look in some haystack of bytes’ is super optimized.

SSC2 and VVX, ARM platform has an assembly impleentation for it, Musil has its own version in C, doesn’t use assembly but it uses C.

So if you have a really rare byte that you feed to that memchar routine, it is going to stay in that memchar routine for the vast majority of that file that you are searching. If you have a very frequent byte that just happens to be the last byte in your ‘needle’, you’ll just be ping-ponging into the memchar routine. You want to spend as much time inside that symd vectorized loop.

 Andrew Callahan had the insight that if you take a background frequency heuristic. Take some common text/source code, ‘hey this byte is not so rae, hey this one is pretty arre’ and instead of always picking the ‘last’ byte you look through the whole needle and pick the one that is probably the rarest, and feed that to memchar, and that’s your ‘skip-loop’, and in the common cases this is where the vast majority of the performance improvements come from comparing ripgrep to grep.

When you have a ‘fast path’ and a ‘slow path’, the optimization can often just be ‘cause as many inputs as possible to just use the fast path’.

[https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html](https://lists.freebsd.org/pipermail/freebsd-current/2010-August/019310.html)

memory maps can be worse than not using memory maps on a whole bunch of really tiny files, they only make a difference when you’re searching huge files, and there’s only a small advantage. Parallelism is both at the search and directory traversal level. 

There’s benefits to parallel directory traversal.

if the entire pattern is just a pattern. ripgrep’s optimizations, if you have a literal at the beginning/end, or some combination in the middle, it’ll detect those and it’ll run a substring search on those and then run regex on just that specific line.

Regex 

- can be done with ‘backtracking’
- Or NFA nondeterministic finite automaton simulation
- Never implemented PCIE like regex engine
- Break a regex down into one of many opcodes, regex executes like a vm,
- Backtracking ones, PCRE has recursion and callouts and all kinds of crazy things, main drawback is that it’s hard to implement those things in a way that is fast from a time complexity point of view.
- It is very easy to create a regex that takes exponential+ time, leads to a whole class of vulns
- REDOS - regex denial of service attacks
- Finite Automaton approach
    - If you implement the standard NFA simulation that you may see in a textbook, great time complexity bound (length of regex * length of input), problem with this approach is that it is dog-slow in practice.
    - This happens to be what GOs regex engine actually implements
        - open issue
    - typical way to speed up is Hybrid NFA-DFA
    - NFA vs DFA, nfa can be in multiple states at once, dfa can be in one state at once.
    - When you see a new byte when you’re in an NFA, for every byte you’re in you need to update every single state. with DFA you have a single byte and a single state, so you can always update with a constant number of instructions and a table in memory.
    - Building a DFA can have a potentially exponential number of states, and especially if you have unicode,
    - Building the regex takes too long, many try to build
    - LazyDFA or hybrid mfa-dfa, it is something that is used in various places, lexcox re-popularized it, it’s how RE2 is fast, fundamental reason that it is fast is because it combines MFA and DFA, it only compiles what it needs for the next byte of input, and then caches what it has compiled. Eventually you get to a point where you’ve cached all the states you’ll ever need, which is typically a very small subset of the total possible states.
    - Failure mode is you have to regenerate the entire state on every byte, there are lots of regexes like this but in practice this ends up being a decent process.
    - You’re actually doing ‘subset construction’ on the fly, this is the process of turning a NFA into a DFA.
    - Don’t use the JustInTime, google’s V8 engine has a JIT, both are impressively fast. They actually compile the regex engine into machine code. They are, at runtime, producing machine code, based on a backtracking regex.

Additional Russ Cox Regex’s are just a treat to read. Read them all.

###vim7.4 comments on release notes.

- What is now called the "old" regexp engine uses a backtracking algorithm. It tries to match the pattern with the text in one way, and when that fails it goes back and tries another way. This works fine for simple patterns, but complex patterns can be very slow on longer text.
- The new engine uses a state machine. It tries all possible alternatives at the current character and stores the possible states of the pattern. This is a bit slower for simple patterns, but much faster for complex patterns and long text.