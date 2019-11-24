---
title: 'Verifying Wasm Functions: `$i64.reverse_bytes`'
author:
-   Rikard Hjort
-   Stephen Skeirik
date: \today
header-includes:
-   \usepackage{amssymb}
-   \newcommand{\K}{$\mathbb{K}$~}
-   \newcommand{\instr}{instr}
-   \newcommand{\STORE}{\textit{S}}
-   \newcommand{\FRAME}{\textit{F}}
-   \newcommand{\CONST}{\texttt{const~}}
-   \newcommand{\DATA}{\texttt{data}}
-   \newcommand{\FUNCS}{\texttt{funcs}}
-   \newcommand{\GLOBALS}{\texttt{globals}}
-   \newcommand{\GROW}{\texttt{grow}}
-   \newcommand{\ITHREETWO}{\texttt{i32}}
-   \newcommand{\LOCALS}{\texttt{locals}}
-   \newcommand{\MAX}{\texttt{max}}
-   \newcommand{\MEMADDRS}{\texttt{memaddrs}}
-   \newcommand{\MEMORY}{\texttt{memory}}
-   \newcommand{\MEMS}{\texttt{mems}}
-   \newcommand{\MEMINST}{\textit{meminst}}
-   \newcommand{\MODULE}{\texttt{module}}
-   \newcommand{\SIZE}{\texttt{size}}
-   \newcommand{\TABLES}{\texttt{tables}}
-   \newcommand{\with}{\text{~with~}}
-   \newcommand{\stepto}{~\hookrightarrow~}
-   \newcommand{\wif}[1]{\text{if}~#1}
-   \newcommand{\diminish}[1]{\begin{footnotesize}#1\end{footnotesize}}
---

This is our journey of verifying a realistic EWasm contract.

In doing so, we want to show you the practicalities of verification work.
We also want to share our latest progress on our tool, K, and show you how to use it.

# How semantics are defined

A K semantics consists of a *syntax* for the language, and a set of *transition rules* over a *state*.
The state consists of *cells* which are written with an XML-like syntax.

## Rules

The bulk of the semantics are the transition rules.
They describe how the state of the program changes gradually through rewrites.
Initially, the state contains of 1) default initial values and 2) the program itself, in the `<k>` cell.
When the program terminates, the program is gone, and the state has changed in some meaningful way.

Here's an example of a simple rule in KWasm.
The Wasm `drop` instruction simply drops the top element on the stack.

```
rule <k> drop => . ... </k>
      <valstack> _ : VALSTACK => VALSTACK </valstack>
```

Here's how you read it:
- The first element on the `<k>` cell (the next instruction) is `drop`, which rewrites to `.`, meaning it is deleted.
  The `...` means there may be more instructions following `drop`.
  This is syntactic sugar for `<k> drop ~> MORE_STUFF => MORE_STUFF <k>`: taking the entire cell, including `drop`, and rewriting the contents to whatever comes after `drop`.
  We introduced another arrow here: `~>`.
  We usually read it as "and then" -- it's just a way to order things in a list-like fashion.
- The stack consists of a top element and a tail, which gets rewritten to the tail.
  Here there is no syntactic sugar going on.

# How the semantics get used for proofs.

<!-- TODO: Rewrite this to be less academic -->
As has been shown, a \K semantics can be read and understood as a computational transition system specifying an interpreter.
But it can also be understood as a logic theory under which we can prove properties about programs.

The set of rewrite rules in KWasm, $\Sigma$, are the axioms of the theory of KWasm transitions, $T$, where $\Sigma \subseteq T$.

The full theory, $T$, consists of all theorems which are provable from the axioms.

To use the \K framework for deductive program verification, one writes proof obligations as regular rewrite rules, which the \K prover tries to prove or disprove belong to $T$.

An implication (rewrite) is a theorem of $T$ iff all terminating paths starting on the left-hand side eventually reach a state that unifies with the right-hand side.
Take, for example, the following implication:

```
rule <k> foo X Y => Z ... </k>
  requires Z >Int X +Int Y
```

This is a theorem iff starting in the configuration `<k> foo X Y ... </k>` (with all cells except the `<k>` cell unspecified), all paths either

1. do not terminate, or
2. end up with an integer on top of the `<k>` cell, everything else that was initially following `foo X Y`, indicated by `...` was left unchanged, and the final integer is larger than the sum of `X` and `Y`.

If, for example, we have the following rules, the spec would be provable:

```
rule [a] <k> foo X Y => foo Y X ... </k>
rule [b] <k> foo X Y => X +Int Y +Int 1 ... </k>
```

Then all paths which eventually apply rule `b` would unify with the right-hand side of the proof obligation.
The path which applies rule `a` forever will never terminate.
This is enough for the spec to pass.

# Warm-up examples

<!-- TODO: Rewrite to be less academic -->

## A Very Simple Proof

A proof obligation in \K is specified exactly like a regular semantic rule.

Just like in a semantic rule, the values mentioned may be symbolic.

A set of these proof obligations is called a *spec*.

Below is a simple spec which asserts that copying the value of a local variable to the stack with `local.get` and then writing that value back to the same variable with `local.set`

1. terminates, as expressed by the whole program rewriting to `.`, and
2. produces no other changes in the state, since there are no other rewrites.

```k
module LOCALS-SPEC
    imports WASM-TEXT
    imports KWASM-LEMMAS

    rule <k> (local.get X:Int) (local.set X:Int) => . ... </k>
         <locals>
           X |-> < ITYPE > VAL
         </locals>

endmodule
```

The program in the `<k>` cell is simple to verify, because during the course of its evaluation, only one semantic rule ever applies at a time.

The prover will first need to apply the statement heating rule followed by parenthesis unpacking

```k
rule <k> (S:Stmt SS) => S ~> SS ... </k>
  requires SS =/=K .Stmts
rule <k> ( PI:PlainInstr ):FoldedInstr => PI ... </k>
```

Now the `<k>` cell of the spec contains

```k
local.get X ~> (local.set X)
```

Next, the rule for `local.get`[^1] applies

```k
rule <k> local.get I:Int => . ... </k>
      <valstack> VALSTACK => VALUE : VALSTACK </valstack>
      <locals> ... I |-> VALUE ... </locals>
```

giving the new configuration (after `.` is removed and parenthesis unpacking)

```k
<k> local.set X:Int ... </k>
<valstack> < ITYPE > VAL : VALSTACK </valstack>
<locals>
  X |-> < ITYPE > VAL
</locals>
```

where `VALSTACK` is whatever the stack contained before.

Lastly, the rule for `local.set`[^2] applies

```k
rule <k> local.set I:Int => . ... </k>
     <valstack> VALUE : VALSTACK => VALSTACK </valstack>
     <locals> ... I |-> (_ => VALUE) ... </locals>
```

giving

```
<k> . ... </k>
<valstack> VALSTACK </valstack>
<locals>
  X |-> < ITYPE > VAL
</locals>
```

which unifies with the right-hand side of the  configuration.

In this simple case we were able to simply state how a program would terminate and leave the state unchanged, and the prover could infer it for us.
Indeed, in making this example, the specification above was written and proved on the first try.
The proving process is often not so straight-forward, however, and may require some tweaking and ingenuity.

## A trickier example

Some proofs require that we further specify our intended semantics and encode the invariants of the transition system.

As an example, we take the exact analogue of our previous proof.
Only this time, instead of modifying local variables we are modifying heap storage.

`#inUnsignedRange` captures the invariants that all integer values, once passed through an `ITYPE.const`, will be represented by their corresponding unsigned value, regardless of signed representation.
I.e., any int value in the state, except at some points in the `<k>` cell, is represented is considered to be in $\mathbb{Z}_{\text{ITYPE}}$, and the representative is chosen to be the value of the class in the range 0 to $2^{|\text{ITYPE}|-1}$.

An invariant the semantics have been designed to maintain (but that we have yet to prove it maintains) is that of `#isByteMap`.

```k
module MEMORY-SPEC
    imports WASM-TEXT
    imports KWASM-LEMMAS

    rule <k> (i32:IValType.store (i32.const ADDR) (i32.load (i32.const ADDR)):Instr):Instr => . ... </k>
         <curModIdx> CUR </curModIdx>
         <moduleInst>
           <modIdx> CUR </modIdx>
           <memAddrs> 0 |-> MEMADDR </memAddrs>
           ...
         </moduleInst>
         <memInst>
           <mAddr> MEMADDR </mAddr>
           <msize> SIZE </msize>
           <mdata> BM   </mdata>
           ...
         </memInst>
       requires #chop(<i32> ADDR) ==K <i32> EA
        andBool EA +Int #numBytes(i32) <=Int SIZE *Int #pageSize()
        andBool #isByteMap(BM)

endmodule
```

This spec will not pass.

The reason is that storing to and reading from memory is more complicated than storing local values.

When a value is stored to memory it is getting spliced into bytes.
When a value is read from memory, the bytes are assembled into a integer value.

Conceptually, the load will put on the stack the following:

$$
val = bm[addr] + (bm[addr + 1] + (bm[addr + 2] + bm[addr + 3] * 256) * 256) * 256
$$

The store operation takes the value off the stack, and conceptually stores the following sequence of bytes:

\begin{align*}
bm[addr]   &:= val \mod 256 \\
bm[addr+1] &:= (val / 256) \mod 256 \\
bm[addr+2] &:= (val / 256^2 ) \mod 256 \\
bm[addr+3] &:= (val / 256^3) \mod 256
\end{align*}

If we plug $val$ into the above equation it becomes clear that the modulus and division operators will cancel out exactly so all we are doing is writing the values in each address back.

This type of reasoning presents a challenge for the \K prover using the current semantics.
The semantics uses pure helper functions, `#setRange` and `#getRange` for writing to and reading from the byte map.
These functions expand to a series of `#set` and `#get`, that do the obvious[^3].

However, Z3 can not reason about these functions in the way we would like without giving full definitions in Z3 of the functions themselves.
Since the getting and setting happens at the \K level while the arithmetic reasoning happens at the Z3 level, we are stuck.
We can remedy this by either extending Z3's reasoning capabilities, or \K's.

In this case, we chose to extend \K.
This simple case could be handled by just adding a high-level trusted lemma:

```k
rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM
```

Indeed, we will resort to this later, when dealing with a symbolic type rather than `i32` or `i64`.

We add the lemmas the following lemmas, which should obviously hold in integer and modular arithmetic[^4].

```k
rule (X *Int N +Int Y) modInt N => Y modInt N
rule (Y +Int X *Int N) modInt N => Y modInt N

rule 0 +Int X => X
rule X +Int 0 => X

rule (Y +Int X *Int N) /Int N => (Y /Int N) +Int X
```

Together, they help eliminate the expressions for assignment to

\begin{align*}
bm[addr]   :=~& bm[addr]     & &        & &\mod{256}                       & &          \\
bm[addr+1] :=~& bm[addr]     & &/ 256   & &\mod{256} + bm[addr + 1]        & &\mod{256} \\
bm[addr+2] :=~& bm[addr]     & &/ 256^2 & &\mod{256} + bm[addr + 1] /256   & &\mod{256} \\
           + ~& bm[addr + 2] & &        & &\mod{256}                       & &          \\
bm[addr+3] :=~& bm[addr]     & &/ 256^3 & &\mod{256} + bm[addr + 1] /256^2 & &\mod{256} \\
            +~& bm[addr + 2] & &/ 256   & &\mod{256} + bm[addr+3]          & &\mod{256}
\end{align*}

We can now make use of the invariant that we claim to maintain for the byte map.
We add the following two lemmas:

```k
rule #get(BMAP, IDX) modInt 256 => #get(BMAP, IDX)
  requires #isByteMap(BMAP)
rule #get(BMAP, IDX) /Int   256 => 0
  requires #isByteMap(BMAP)
```

They state that as long as a byte maintains its intended invariant -- that all values are integers from 0 to 255, inclusive -- we may discard the modulus on the values, and the division amount to zeroing them.
The lemma in itself is self-evident since it assumes the byte map maintains the invariant.
The claim that our semantics maintain this invariant is, at present, a conjecture.

With all these lemmas added to the theory, the proof goes through.

### Using a Symbolic Type

If we want to make a proof that uses a symbolic type, rather than `i32` or `i64`, matters become more complicated.
Without knowing the type, `#setRange` and `#getRange` will receive a symbolic `WIDTH` argument, and not be able to expand.

To make a proof like that go through, we introduce the higher level idempotence lemma from before:

```k
rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM
```

But rather than including it in the set of manual axioms for all our verification, we can apply it locally, only where it is needed.

```k
require "kwasm-lemmas.k"

module MEMORY-SYMBOLIC-TYPE-LEMMAS
  imports KWASM-LEMMAS

  rule #setRange(BM, EA, #getRange(BM, EA, WIDTH), WIDTH) => BM

endmodule

module MEMORY-SYMBOLIC-TYPE-SPEC
    imports WASM-TEXT
    imports MEMORY-SYMBOLIC-TYPE-LEMMAS

    rule <k> (ITYPE:IValType.store (i32.const ADDR) (ITYPE.load (i32.const ADDR)):Instr):Instr => . ... </k>

...
```

By invoking the \K prover with the option `--def-module MEMORY-SYMBOLIC-TYPE-LEMMAS` instead of the usual `--def-module KWASM-LEMMAS`, the prover will accept this new lemma in to its axioms, and the proof will go through.



[^1]: The rule is paraphrased here, it actually is slightly more complex to deal with identifiers.
[^2]: Again, paraphrased.
[^3]: Actually, there is one non-obvious part of each function: when the stored value is 0, that is represented by no entry.
      The two functions respect that by erasing 0-valued entries and interpreting an empty entry as 0, respectively.
[^4]: N.B. that we cannot use the distributive property of division, as that holds over the rational numbers and its supersets, not over the integers.




<!--- What happens when do the concrete type proof, with, with different lemmas added.

No lemmas:

#set(
#set(
#set(
#set(BM

, EA,
_modInt_((#get(BM, EA) + ((#get(BM, (EA + 1)) + ((#get(BM, ((EA + 1) + 1)) + ((#get(BM, (((EA + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)), 256))

, (EA + 1),
_modInt_(((#get(BM, EA) + ((#get(BM, (EA + 1)) + ((#get(BM, ((EA + 1) + 1)) + ((#get(BM, (((EA + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256), 256))

, ((EA + 1) + 1),
_modInt_((((#get(BM
, EA) + ((#get(BM, (EA + 1)) + ((#get(BM, ((EA + 1) + 1)) + ((#get(BM, (((EA + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256) / 256), 256)),

(((EA + 1) + 1) + 1),
_modInt_(((((#get(BM, EA) + ((#get(BM, (EA + 1)) + ((#get(BM, ((EA + 1) + 1)) + ((#get(BM, (((EA + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256) / 256) / 256), 256))




Then added:
rule (Y +Int X *Int N) modInt N => Y modInt N [simplification]



#set(
#set(
#set(BM

, (ADDR + 1),
_modInt_(((#get(BM, ADDR) + ((#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256), 256))

, ((ADDR + 1) + 1),
_modInt_((((#get(BM, ADDR) + ((#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256) / 256), 256))

, (((ADDR + 1) + 1) + 1),
_modInt_(((((#get(BM, ADDR) + ((#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256)) * 256)) / 256) / 256) / 256), 256))



Then added:
rule (Y +Int X *Int N) /Int N => (Y /Int N) +Int X [simplification]





#set(
#set(
#set(BM

, (ADDR + 1),
_modInt_((0 + (#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256))), 256))

, ((ADDR + 1) + 1), _modInt_(((0 + (#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256))) / 256), 256))

, (((ADDR + 1) + 1) + 1),
_modInt_((((0 + (#get(BM, (ADDR + 1)) + ((#get(BM, ((ADDR + 1) + 1)) + ((#get(BM, (((ADDR + 1) + 1) + 1)) + 0) * 256)) * 256))) / 256) / 256), 256))


Then added:
rule 0 +Int X => X [simplification]


#set(BM, (((ADDR + 1) + 1) + 1), _modInt_((#get(BM, (((ADDR + 1) + 1) + 1)) + 0), 256))

rule X +Int 0 => X [simplification]



Also needed somewhere:
rule #get(BMAP, IDX) modInt 256 => #get(BMAP, IDX)
  requires #isByteMap(BMAP)
rule #get(BMAP, IDX) /Int   256 => 0
  requires #isByteMap(BMAP)

-->

# WRC20 Verification: first step

WRC20 is a Wasm version of an ERC20.
The WRC20 program can be found [here](https://gist.github.com/poemm/823566e89def007daa5d97ac5bd14419).
It's simpler than an ERC20: it only has a function for transferring (the caller's) funds to another address, and a function for querying the balance of any address.
Keep in mind also that Ewasm is part of Ethereum 2.0, phase 2.
It's still a work in progress, so exactly what Ewasm will look like is unclear.
This is based on an early work-in-progress specification of Ewasm.

In the end, we want to verify the behavior of the two external functions, `transfer` and `getBalance`.
To do so we will need to introduce K rules about the Ethereum state.
That's a topic for the next blog post.
This time, we will focus on a helper function: `$i64.reverse_bytes`.
This function does a simple thing: it takes an `i64` as a parameter and returns the `i64` you get by reversing the order of the bytes.

```
x_0 x_1 x_2 x_3 x_4 x_5 x_6 x_7
```

becomes

```
x_7 x_6 x_5 x_4 x_3 x_2 x_1 x_0
```

What's the point of this?
Wasm stores numbers to memory as *little endian*, meaning the least significant byte goes in the lowest memory slot.
So for example, storing the `i64` value `1` to memory address 0 means that the memory looks like this

```
Address: 0        1        2        3        4        5        6        7
Value:   00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

Ewasm call data is by convention interpreted as *big endian*, meaning the bytes are in the opposite order.
So the contract must reverse all integer balances: in `transfer`, it must reverse the amount to be sent, and in `getBalance` it must reverse the result before returning.

Without further ado, here is what we are going to try to prove[^5]:

```
    rule <k> #wrc20ReverseBytes // A macro. Expands to the function defintion.
          ~> (i32.const ADDR)
             (i32.const ADDR)
             (i64.load)
             ( invoke NEXTADDR ) // Invoke is an internal Wasm command, similar to `call`.
             (i64.store)
          => .
             ...
        </k>
        <curModIdx> CUR </curModIdx>
        <moduleInst>
          <modIdx> CUR </modIdx>
          <memAddrs> 0 |-> MEMADDR </memAddrs>
          <types> TYPES => _ </types>                                     /* These five state changes  */
          <nextTypeIdx> NEXTTYPEIDX => NEXTTYPEIDX +Int 1 </nextTypeIdx>  /* are due to the fact that  */
          <funcIds> _ => _ </funcIds>                                     /* we declare a new function */
          <funcAddrs> _ => _ </funcAddrs>                                 /* in the first step of the  */
          <nextFuncIdx> NEXTFUNCIDX => NEXTFUNCIDX +Int 1 </nextFuncIdx>  /* specification.            */
          ...
        </moduleInst>
        <funcs> _ => _ </funcs>                                      /* So is this change. */
        <nextFuncAddr> NEXTADDR => NEXTADDR +Int 1 </nextFuncAddr>   /* And this one.      */
        <memInst>
          <mAddr> MEMADDR  </mAddr>
          <msize> SIZE     </msize>
          <mdata> BM => BM' </mdata>
          ...
        </memInst>
      requires notBool unnameFuncType(asFuncType(#wrc20ReverseBytesTypeDecls)) in values(TYPES)
       andBool ADDR +Int #numBytes(i64) <=Int SIZE *Int #pageSize()
       andBool #isByteMap(BM)
       andBool #inUnsignedRange(i64, X)
       andBool #inUnsignedRange(i32, ADDR)
      ensures  #get(BM, ADDR +Int 0) ==Int #get(BM', ADDR +Int 7 )
       andBool #get(BM, ADDR +Int 1) ==Int #get(BM', ADDR +Int 6 )
       andBool #get(BM, ADDR +Int 2) ==Int #get(BM', ADDR +Int 5 )
       andBool #get(BM, ADDR +Int 3) ==Int #get(BM', ADDR +Int 4 )
       andBool #get(BM, ADDR +Int 4) ==Int #get(BM', ADDR +Int 3 )
       andBool #get(BM, ADDR +Int 5) ==Int #get(BM', ADDR +Int 2 )
       andBool #get(BM, ADDR +Int 6) ==Int #get(BM', ADDR +Int 1 )
       andBool #get(BM, ADDR +Int 7) ==Int #get(BM', ADDR +Int 0 )
```

The interesting parts are:
- the `<k>` cell
- the `<memInst>` cell group, and
- the pre- and post-conditons, `requires` and `ensures`.

[^5]: We should note that this is slightly different from the lemma we will need in the end when verifying the `transfer` and `getBalance` functions.
  To make this spec useful, we need to make sure the starting state of the spec matches the state of the program when `transfer` and `getBalance` calls `$i64.reverse_bytes`.
  If we do that, the prover will be able to go: "Aha! This state corresponds exactly to something I've proven, I can just jump to the conclusion!"
  But the above version makes for nice presentation.