WebAssembly Text Format
=======================

```k
require "wasm.k"
require "data.k"

module WASM-TEXT-SYNTAX
    imports WASM-TEXT
    imports WASM-SYNTAX
endmodule

module WASM-SYNTAX
    imports WASM-TOKEN-SYNTAX
    imports WASM-DATA
endmodule
```

Wasm Tokens
-----------

`WASM-TOKEN-SYNTAX` module defines the tokens used in parsing programs.

```k
module WASM-TOKEN-SYNTAX
```

### Strings

In WebAssembly, strings are defined differently to K's built-in strings, so we have to write the definition of WebAssembly `WasmString` in a separate module, and use the module just for parsing the program.
Note that you cannot use a normal K `String` in any production definitions, because the definitions of `String` and `WasmString` overlap, and the K tokenizer does not support ambiguity.

```k
    syntax WasmStringToken ::= r"\\\"(([^\\\"\\\\])|(\\\\[0-9a-fA-F]{2})|(\\\\t)|(\\\\n)|(\\\\r)|(\\\\\\\")|(\\\\')|(\\\\\\\\)|(\\\\u\\{[0-9a-fA-F]{1,6}\\}))*\\\"" [token]
 // -----------------------------------------------------------------------------------------------------------------------------------------------------------------------
```

### Identifiers

In WebAssembly, identifiers are defined by the regular expression below.

```k
    syntax IdentifierToken ::= r"\\$[0-9a-zA-Z!$%&'*+/<>?_`|~=:\\@^.-]+" [token]
 // ----------------------------------------------------------------------------
```

### Integers

In WebAssembly, integers could be represented in either the decimal form or hexadecimal form.
In both cases, digits can optionally be separated by underscores.

```k
    syntax WasmIntToken ::= r"[\\+-]?[0-9]+(_[0-9]+)*"               [token]
                          | r"[\\+-]?0x[0-9a-fA-F]+(_[0-9a-fA-F]+)*" [token]
 // ------------------------------------------------------------------------
```

### Layout

WebAssembly allows for block comments using `(;` and `;)`, and line comments using `;;`.
Additionally, white-space is skipped/ignored.
Declaring regular expressions of sort `#Layout` infroms the K lexer to drop these tokens.

```k
    syntax #Layout ::= r"\\(;([^;]|(;+([^;\\)])))*;\\)" [token]
                     | r";;[^\\n\\r]*"                  [token]
                     | r"[\\ \\n\\r\\t]"                [token]
 // -----------------------------------------------------------
```

```k
endmodule
```

Wasm Textual Format
-------------------

```k
module WASM-TEXT
    imports WASM
```

The text format is a concrete syntax for Wasm.
It allows specifying instructions in a folded, S-expression like format, and a few other syntactic sugars.
Most instructions, those in the sort `PlainInstr`, have identical keywords in the abstract and concrete syntax, and can be used idrectly.

### Text Integers

All integers given in the text format are automatically turned into regular integers.
That means converting between hexadecimal and decimal when necessary, and removing underscores.

**TODO**: Symbolic reasoning for sort `WasmIntToken` not tested yet. In the future should investigate which direction the subsort should go. (`WasmIntToken` under `Int`/`Int` under `WasmIntToken`)

```k
    syntax WasmInt ::= Int
    syntax WasmInt ::= WasmIntToken [klabel(WasmInt), avoid, symbol, function]
 // --------------------------------------------------------------------------
    rule `WasmInt`(VAL) => WasmIntToken2Int(VAL)

    syntax String ::= WasmIntToken2String    ( WasmIntToken ) [function, functional, hook(STRING.token2string)]
    syntax Int    ::= WasmIntTokenString2Int ( String       ) [function]
                    | WasmIntToken2Int       ( WasmIntToken ) [function]
 // --------------------------------------------------------------------
    rule WasmIntTokenString2Int(S)  => String2Base(replaceFirst(S, "0x", ""), 16) requires findString(S, "0x", 0) =/=Int -1
    rule WasmIntTokenString2Int(S)  => String2Base(                        S, 10) requires findString(S, "0x", 0)  ==Int -1

    rule WasmIntToken2Int(VAL) => WasmIntTokenString2Int(replaceAll(WasmIntToken2String(VAL), "_", ""))
```

### Folded Instructions

Folded instructions are a syntactic sugar where expressions can be grouped using parentheses for higher readability.

```k
    syntax Instr ::= FoldedInstr
 // ----------------------------
```

One type of folded instruction are `PlainInstr`s wrapped in parentheses and optionally includes nested folded instructions to indicate its operands.

```k
    syntax FoldedInstr ::= "(" PlainInstr Instrs ")"
                         | "(" PlainInstr        ")" [prefer]
 // ---------------------------------------------------------
    rule <k> ( PI:PlainInstr IS:Instrs ):FoldedInstr => IS ~> PI ... </k>
    rule <k> ( PI:PlainInstr           ):FoldedInstr =>       PI ... </k>
```

Another type of folded instruction is control flow blocks wrapped in parentheses, in which case the `end` keyword is omitted.

```k
    syntax FoldedInstr ::= "(" "block" OptionalId TypeDecls Instrs ")"
 // ------------------------------------------------------------------
    rule <k> ( block               TDECLS:TypeDecls INSTRS:Instrs ) => block    TDECLS INSTRS end ... </k>
    rule <k> ( block ID:Identifier TDECLS:TypeDecls INSTRS:Instrs ) => block ID TDECLS INSTRS end ... </k>

    syntax FoldedInstr ::= "(" "if" OptionalId TypeDecls Instrs "(" "then" Instrs ")" ")"
                         | "(" "if" OptionalId TypeDecls Instrs "(" "then" Instrs ")" "(" "else" Instrs ")" ")"
 // -----------------------------------------------------------------------------------------------------------
    rule <k> ( if OID:OptionalId TDECLS:TypeDecls C:Instrs ( then IS ) )              => C ~> if OID TDECLS IS          end ... </k>
    rule <k> ( if                TDECLS:TypeDecls C:Instrs ( then IS ) ( else IS' ) ) => C ~> if     TDECLS IS else IS' end ... </k>
    rule <k> ( if  ID:Identifier TDECLS:TypeDecls C:Instrs ( then IS ) ( else IS' ) ) => C ~> if  ID TDECLS IS else IS' end ... </k>

    syntax FoldedInstr ::= "(" "loop" OptionalId TypeDecls Instrs ")"
 // -----------------------------------------------------------------
    rule <k> ( loop               TDECLS:TypeDecls IS ) => loop    TDECLS IS end ... </k>
    rule <k> ( loop ID:Identifier TDECLS:TypeDecls IS ) => loop ID TDECLS IS end ... </k>
```

### Identifiers

When we want to specify an identifier, we can do so with the following helper function.

```k
    syntax IdentifierToken ::= String2Identifier(String) [function, functional, hook(STRING.string2token)]
```

### Looking up Indices

In the abstract Wasm syntax, indices are always integers.
In the text format, we extend indices to incorporate identifiers.
We also enable context lookups with identifiers.

```k
    syntax Index ::= Identifier
 // ---------------------------
    rule #ContextLookup(IDS:Map, ID:Identifier) => {IDS [ ID ]}:>Int
      requires ID in_keys(IDS)
```

### Block Instructions

In the text format, block instructions can have identifiers attached to them, and branch instructions can refer to these identifiers.
The `<labelIds>` cell maps labels to the depth at which they occur.
To ensure the bookkeeping mapping in `<labelIds>` is properly updated when branching, we don't make a new `Label` production, but instead a different one: `IdLabel`.

Branching with an identifier is the same as branching to the label with that identifier.
The correct label index is calculated by looking at whih depth the index occured and what depth execution is currently at.

```k
    rule <k> br ID:Identifier => br DEPTH -Int DEPTH' -Int 1 ... </k>
         <labelDepth> DEPTH </labelDepth>
         <labelIds> ... ID |-> DEPTH' ... </labelIds>
```

Finally, we introduce the text format block instructions, which may have identifiers after each keyword.
If more than one identifier is present, they all have to agree (they are just there to make clear what if-block they belong to).
If identifiers are used, one must occur after the initial keyword (`block`, `if` or `loop`).

```k
    syntax Instr ::= BlockInstr
 // ---------------------------

    syntax BlockInstr ::= "block" Identifier TypeDecls Instrs "end" OptionalId
 // --------------------------------------------------------------------------
    rule <k> block ID:Identifier TDECLS IS end OID':OptionalId => block TDECLS IS end ... </k>
         <labelDepth> DEPTH </labelDepth>
         <labelIds> IDS => IDS[ID <- DEPTH] </labelIds>
      requires ID ==K OID'
        orBool notBool isIdentifier(OID')

    syntax BlockInstr ::= "loop" Identifier TypeDecls Instrs "end" OptionalId
 // -------------------------------------------------------------------------
    rule <k> loop ID:Identifier TDECLS:TypeDecls IS end OID':OptionalId => loop TDECLS IS end ... </k>
         <labelDepth> DEPTH </labelDepth>
         <labelIds> IDS => IDS[ID <- DEPTH] </labelIds>
      requires ID ==K OID'
        orBool notBool isIdentifier(OID')
```

In the text format, it is also allowed to have a conditional without the `else` branch.

```k
    syntax BlockInstr ::= "if" Identifier TypeDecls Instrs "else" OptionalId Instrs "end" OptionalId
                        | "if" OptionalId TypeDecls Instrs                          "end" OptionalId
 // ------------------------------------------------------------------------------------------------
    rule <k> if TDECLS:TypeDecls IS end => if TDECLS IS else .Instrs end ... </k>

    rule <k> if ID:Identifier TDECLS:TypeDecls IS end OID'':OptionalId => if ID TDECLS IS else ID .Instrs end ID ... </k>
      requires ID ==K OID''
        orBool notBool isIdentifier(OID'')

    rule <k> if ID:Identifier TDECLS:TypeDecls IS else OID':OptionalId IS' end OID'':OptionalId => if TDECLS:TypeDecls IS else IS' end ... </k>
         <labelDepth> DEPTH </labelDepth>
         <labelIds> IDS => IDS[ID <- DEPTH] </labelIds>
      requires ( ID ==K OID'  orBool notBool isIdentifier(OID')  )
       andBool ( ID ==K OID'' orBool notBool isIdentifier(OID'') )
```

### Memory and Tables

Intitial memory data, and initial table elements can be given inline in the text format.
We supply macros that will unfold these definitions when they are part of a body of definitions, such as in a module.
This is to ensure that the unfolding happens before the different elements in a module are grouped together, so as to maintain their order.

We also supply rules for when the inlined definitions are encountered on top of the `<k>` cell, so that they can be desugared on the fly.

This is useful when specifying modules in the more lax KWasm format, where they can be declared as they are needed.

```k
    syntax Identifier ::= ".MemoryIdentifier" | ".TableIdentifier"

    syntax MemorySpec ::= "(" "data" DataString ")"
 // -----------------------------------------------
    rule ( memory ( data DS ) ) => ( memory .MemoryIdentifier (data DS) ) [macro]

    rule ( memory ID:Identifier ( data DS ) ) DEFS:Defns
      => ( memory ID #lengthDataPages(DS) #lengthDataPages(DS) ):MemoryDefn
         ( data   ID (offset (i32.const 0) .Instrs) DS )
         DEFS
      [macro]

    rule <k> ( memory ID:Identifier ( data DS ) )
          => ( memory ID #lengthDataPages(DS) #lengthDataPages(DS) ):MemoryDefn
          ~> ( data   ID (offset (i32.const 0) .Instrs) DS )
          ...
         </k>

    syntax TableSpec ::= TableElemType "(" "elem" ElemSegment ")"
 // -------------------------------------------------------------
    rule ( table funcref ( elem ES ) ) => ( table .TableIdentifier funcref (elem ES) ) [macro]

    rule ( table ID:Identifier funcref ( elem ES ) ) DEFS:Defns
      => ( table ID #lenElemSegment(ES) #lenElemSegment(ES) funcref ):TableDefn
         ( elem  ID (offset (i32.const 0) .Instrs) ES )
         DEFS
      [macro]

    rule <k> ( table ID:Identifier funcref ( elem ES ) )
          => ( table ID #lenElemSegment(ES) #lenElemSegment(ES) funcref ):TableDefn
          ~> ( elem  ID (offset (i32.const 0) .Instrs) ES )
          ...
         </k>
```

### Exports

Exports can be declared like regular functions, memories, etc., by giving an inline export declaration.
In that case, it simply desugars to the definition followed by an export of it.
If no identifer is present, one must be introduced so that the export can refer to it.
Note that it is possible to define multiple exports inline, i.e. export a single entity under many names.

```k
    syntax InlineExport  ::= "(" "export" WasmString ")"
 // ----------------------------------------------------

    syntax GlobalSpec ::= InlineExport GlobalSpec
 // ---------------------------------------------
    rule <k> ( global                  EXPO:InlineExport SPEC:GlobalSpec )
          => ( global #freshId(NEXTID) EXPO              SPEC            )
          ...
         </k>
         <nextFreshId> NEXTID => NEXTID +Int 1 </nextFreshId>

    rule <k> ( global ID:Identifier ( export ENAME ) SPEC:GlobalSpec )
          => ( export ENAME ( global ID ) )
          ~> ( global ID SPEC )
          ...
         </k>

    syntax FuncSpec   ::= InlineExport FuncSpec
 // -------------------------------------------
    rule <k> ( func                  EXPO:InlineExport SPEC:FuncSpec )
          => ( func #freshId(NEXTID) EXPO              SPEC          )
          ...
         </k>
         <nextFreshId> NEXTID => NEXTID +Int 1 </nextFreshId>

    rule <k> ( func ID:Identifier ( export ENAME ) SPEC:FuncSpec )
          => ( export ENAME ( func ID ) )
          ~> ( func ID SPEC )
          ...
         </k>

    syntax TableSpec  ::= InlineExport TableSpec
 // --------------------------------------------
    rule <k> ( table                  EXPO:InlineExport SPEC:TableSpec )
          => ( table #freshId(NEXTID) EXPO              SPEC           )
          ...
         </k>
         <nextFreshId> NEXTID => NEXTID +Int 1 </nextFreshId>

    rule <k> ( table ID:Identifier ( export ENAME ) SPEC:TableSpec )
          => ( export ENAME ( table ID ) )
          ~> ( table ID SPEC )
          ...
         </k>

    syntax MemorySpec ::= InlineExport MemorySpec
 // ---------------------------------------------
    rule <k> ( memory                  EXPO:InlineExport SPEC:MemorySpec )
          => ( memory #freshId(NEXTID) EXPO              SPEC            )
          ...
         </k>
         <nextFreshId> NEXTID => NEXTID +Int 1 </nextFreshId>

    rule <k> ( memory ID:Identifier ( export ENAME ) SPEC:MemorySpec )
          => ( export ENAME ( memory ID ) )
          ~> ( memory ID SPEC )
          ...
         </k>
```

### Imports

Imports can be declared like regular functions, memories, etc., by giving an inline import declaration.

```k
    syntax InlineImport ::= "(" "import" WasmString WasmString ")"
 // --------------------------------------------------------------

    syntax GlobalSpec ::= InlineImport TextFormatGlobalType
    syntax FuncSpec   ::= InlineImport TypeUse
    syntax TableSpec  ::= InlineImport TableType
    syntax MemorySpec ::= InlineImport MemType
 // ------------------------------------------
    rule ( global OID:OptionalId (import MOD NAME) TYP          ) => ( import MOD NAME (global OID TYP ) )  [macro]
    rule ( func   OID:OptionalId (import MOD NAME) TUSE         ) => ( import MOD NAME (func   OID TUSE) )  [macro]
    rule ( table  OID:OptionalId (import MOD NAME) TT:TableType ) => ( import MOD NAME (table  OID TT  ) )  [macro]
    rule ( memory OID:OptionalId (import MOD NAME) MT:MemType   ) => ( import MOD NAME (memory OID MT  ) )  [macro]
```

```k
endmodule
```
