# Structural guards

Open when writing or changing a guard script (layering, cycles).

- A guard has no allowed list of known breaks. Break the known violations first, then make the rule strict. An allowed list hides growth.
- Each guard script has its own small test script: throwaway source trees, exact exit code and failure lines per case, plus a mutation check.
- A rule that scans the source tree for a forbidden pattern (for example "no X written inline in component files") is a structure rule. It goes in the guard script and fails closed, not in a unit test.
- Never give such a check a count floor ("at least N files use X"). It fails as soon as code is legitimately removed, and it proves nothing.
