
Eventually I will write some real documentation, for now here are a few
notes.

* No free()ing of heap allocated objects, since that's a pain and it's
  not necessary in this case because the amount of data allocated is
  proportional to the size of the (generally small) lua program that
  is parsed.

* "dofile 'filename'" statements expanded inline at global scope. other
  dofiles (e.g. in expressions or inner scope) are not expanded since
  they might be conditional.
