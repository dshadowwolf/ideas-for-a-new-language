###Syntax Description
```
compilation-unit: prologue object | object ;
prologue: imports ;
imports: import | import ',' imports ;
import: IMPORT identifier FROM identifier AS identifier EOL ;
object: object-header { object-body } ;
object-header: OBJECT identifier | OBJECT identifier inheritance ;
inheritance: <currently undefined> ;
object-body: members;
member-variable: [visibility] [storage-spec] typespec identifier ASSIGN [value | identifier] EOL |
                 [visibility] [storage-spec] typespec identifer EOL ;
member-function: [visibility] FUNCTION identifier parameter-list : return-spec { expressions } EOL ;
member: member-variable | member-function ;
members: member | members ;
visibility: <currently undefined> ;
storage-spec: <currently undefined> ;
parameter-list: '('[any whitespace]')' |
  '(' plist ')' ;
plist: plist-item |
  plist-item ',' plist ;
plist-item: typespec identifier |
  typespec identifer ASSIGN constant ;
return-spec: typespec ;
expressions: all-expr |
             all-expr EOL expressions ;
all-expr: math-expr |
          compare-expr |
          call-expr |
          control-expr |
		  loop-expr |
		  lambda-expr |
		  anon-expr |
		  comp-expr |
		  pod-expr |
		  post-expr |
		  other-expr ;
any-expr: math-expr |
          compare-expr |
		  call-expr |
		  assign-op |
		  ( any-expr ) ;
math-expr: any-expr MATHOP any-expr ;
compare-expr: any-expr COMPAREOP any-expr ;
call-expr: identifier ( idclist ) ;
assign-op: identifier ASSIGN any-expr ;
control-expr: if-else |
  switch-case ;
loop-expr: do-while |
  while |
  for |
  for-each ;
lambda-expr: (idlist) => all-expr ;
anon-expr: function parameter-list { expressions } ;
comp-expr: '[' ... ']'  ;
pod-op: identifier |
        constant |
		'[' idclist ']' |
		object-literal ;
post-expr: call-expr limited-binop allowed-postexprs EOL ;
other-expr: assign-op |
            return-op ;
return-op: RETURN value EOL ;
idclist: idc |
         idc ',' idclist ;
idc: identifer |
     constant ;
idlist: identifier |
        identifer ',' idlist ;
object-literal: <currently undefined> ;
limited-binop: BINAND | BINOR ;
allowed-postexprs: call-expr |
                   return-op ;
<NOTE: Unfished, left in this state on purpose. I am not familiar enough with BNF or ABNF to do this properly>
```

###Notes
1. This is in a mix of classical BNF, YACC/Bison style BNF and several other forms. Help is requested to get this into some canonical form.
2. Items in caps are either key-words or operators - for instance 'EOL' is 'END OF LINE' - that is, the semi-colon. Some of them represent multiple possible items - such as 'MATHOP' representing all mathematical operators.
3. Those marked '<currently undefined>' are things that should probably exist - or just plain are required to exist - but have no proposals for them yet.
4. I am actually unsure how to properly define the 'comp-expr' (aka: list comprehension expression) part, as it is a rather complex and tricky part of the language that takes full advantage of the "everything is an expression" nature.
5. An object-literal is a way of defining a complex language object in such a manner that the compiler can automagically build the object as a constant. JSON is technically the Javascript object-literal notation and, in C, something like:
```C
complex_structure_t data = { .member_a = 100; .member_b = "an example"; };
```
Is also an object literal.
6. As needed - or as a need is found - items can be added to the allowed-postexprs list, but this should be limited and never include anything close to all possible expressions. This feature is meant for giving expressiveness, not for allowing programmers to show how "cute" they can be.
