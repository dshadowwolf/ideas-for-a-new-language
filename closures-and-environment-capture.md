# Closures, Captures and Scope

I have recently been working on trying to implement `Turned-C` as a full 
language and realized that I never successfully managed to define how a 
functions execution scope is defined or captured for the "reparenting" 
process. Part of this becomes simple with the introduction of a keyword:
`parent` which acts to represent the "full scope" of the definition site.
But this still leaves some level of work needing done, as just doing, for
instance:
```
define x: integer <- 10;

define movable: function <-> (context: &parent) -> {
  context.x + 10;
};
```
Will leave a lot of questions unanswered as all it shows is that the 
defining context is "captured" as a reference in the "data scope" handed
to the function as part of the evaluation. That it requires a manual
definition of things is part of the issue -- the manual definition
requirement means that you could not, easily, "reparent" any random
function which means every API function that you might be developing and 
using unit tests to "confirm behavior" for needs that special definition.

Instead I think a slightly better measure might be for there to be an 
"implicit" context capture that can be "explicitly exposed" via properly
annotating the function, say something like:
```
define x: integer <- 10;

@[ capture: context ]@
define movable: function <-> () -> {
	context.x + 10;
};

define movable-two: function <-> () -> {
	x + 10;
};
```
In this way we can treat the "captured" context either explicitly or
implicitly and have no need for "extra language" that isn't already part
of the design of Turned-C. Further, we could even extend it to allow for
some "contractual enforcement" such as:
```
define x: integer <- 10;

@[ requires: capture, required-items: [x: integer] ]@
define movable: function <-> () -> {
	x + 10;
};
```
In doing this we somewhat abuse the "annotations parser" by having a
set of "keywords" that define what is actually required in the "new" 
execution context as well as allowing the existing "definition site"
capture to remain in effect if the new context doesn't have any single
item of the requirements.

This leaves just the rules of how things are resolved -- the "shadow
copy" that is either explicitly (via the annotation) or implicitly 
created (via a direct rebinding attempt) is actually the last thing to
be looked at. Instead it should be resolved in a simple pattern of 4
steps:
	1) Check the local scope first -- the referenced variable may have
	been defined as part of function itself.
	2) Check the "bound scope" -- the `self`/`this` keyword is always
	going to refer to the immediately accessible "scope" that comes as
	part of the "parameters" even though it is implicitly there and not
	required to be explicitly listed.
	3) Check the immediate parent context to see if it provides the
	required names and that they have the referenced types (or a close
	relative of the type)
	4) As the final step, go to the shadow-copy and get the value
	required.
