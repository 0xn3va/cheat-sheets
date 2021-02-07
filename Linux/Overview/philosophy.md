The philosophy behind UNIX:
- Write programs that do one thing and do it well
- Write programs to work together (no extra output, don't insist on interactive input)
- Write programs to handle text streams, because that is a universal interface

UNIX also embraced the ["Worse is better"](https://en.wikipedia.org/wiki/Worse_is_better) philosophy.

This thinking is powerful. On a higher level, you can see it a lot in functional programming: build atomic functions that focus on one thing, no extra output, and then compose them together to do complicated things; all functions in the composition are pure; no global variables to keep track of.

Perhaps as a direct result, the design of UNIX focuses on two major components:
- Processes
- Files

{% hint style="info" %}
Everything in UNIX is either a process or a file. Nothing else.
{% endhint %}

# References

- [How Unix Works: Become a Better Software Engineer](https://neilkakkar.com/unix.html)