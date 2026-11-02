# Structural guards

Open when writing or changing a guard script (layering, cycles).

- A guard has no allowed list of known breaks. Break the known violations first, then make the rule strict. An allowed list hides growth.
- Each guard script has its own small test script: throwaway source trees, exact exit code and failure lines per case, plus a mutation check.
