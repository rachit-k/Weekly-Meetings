# Algebraic specification of stateful libraries with Trace-based Reasoning backends

Stateful library specifications can have two kinds of views: a trace view (what HAT uses — finite sequences of events with temporal logic over them) and an algebraic view (constructors, observers, equational laws). HAT's machinery makes the trace view the implementation (decidable inclusion via SFA), but the algebraic view is what humans want to write and reason about. Stateful libraries are naturally specified algebraically — as equational theories of constructors and observers, with invariants as state predicates.  For example: A developer thinking about a `Set` thinks in equations:
```
   mem(insert(s, x), x)       = true
   insert(insert(s, x), x)    = insert(s, x)
   x != y => mem(insert(s, x), y) = mem(s, y)
```
ie, constructors that build state, observers that read it, and laws governing how they interact. Existing trace-based systems such as HAT can verify symbolic temporal specifications over library-interface traces, including specifications involving ghost variables and symbolic predicates. However, their specification languages are low-level and automata-oriented, making specifications difficult to write and reason about. The same "every key has a distinct value" property that's one equation algebraically becomes:

```
   □ ( ⟨put k v = ν | v = el⟩  ⟹  ○ □ ¬⟨put k' v' = ν | v' = el⟩ )
```
an LTLf formula over a trace of events. 

Algebraic systems such as KAT and KMT provide nice and clean equational reasoning, but they do not directly target symbolic trace specifications of black-box stateful libraries with first-class ghost forms and event predicates. KAT/KMT provide an algebraic style of reasoning, where programs/specifications are terms and verification can be phrased as equations or inclusions. That style is much nicer: specs can be composed, simplified, normalized, and reasoned about equationally. KMT can encode client theories, but then ghost state and symbolic predicates tend to be pushed into the theory, rather than being first-class trace/specification constructs. They have the right "algebraic frontend, decidable backend" pattern but at the wrong granularity (programs, not data types). We want the usability and compositionality of algebraic specifications, but the semantic target should remain symbolic traces of black-box library interfaces.
We want to design an algebraic specification language for stateful library interfaces where terms are easy to write and reason about equationally, but their semantics is still given by symbolic traces with ghost state and predicates, so that verification reduces to trace-language inclusion.

### Why not KAT:

First, library behavior is trace-like. A key-value store, file API, iterator, lock, database transaction system, or resource protocol is often specified by the sequence of calls it permits or guarantees. For example, after put(k,v), a later get(k) should return v unless another write intervenes. This is not just a local pre/postcondition; it is a property of interaction history.

Second, realistic trace specs need symbolic structure. We do not want an alphabet like:
```
put_a_0, put_a_1, get_a_0, get_a_1, ...
```
We want symbolic events:
```
put(k,v)
get(k,v)
delete(k)
```
and predicates such as:
```
Has(k,v)
Present(k)
LatestWrite(k,v)
```
These predicates may refer to ghost state, abstract maps, or history summaries. This is exactly where HAT-style symbolic predicates and ghost variables become useful.

### Why not HAT:

Temporal specifications are painful as the user-facing language. HAT can express many of these things, but the user has to manually encode the automaton/temporal structure, representation invariant, event predicates, and ghost variables. That makes specifications hard to write, hard to debug, and hard to manipulate algebraically.

The temporal form has bad algebraic properties — no clean way of substituting equals, composing two libraries' invariants, or swapping a library underneath a client. It also forces temporal reasoning where developers think structurally — "globally, between successive insertions, membership holds". Also, it forces us to write the property while describing it's compilation/recognition in some sense, as in describing the property as a kind of event stream. There's a distinction between describing a property and describing how to recognise it. A specification language should let you say what the property is instead of making you specify the mechanism that detects it. For example, take a library door with open, close, and is_open methods. is_open's specification is simply: is_open(open(s)) = true; is_open(close(s)) = false. But in temporal world, we need to soeficy the mechanism like "some open happened, and no close happened after it."


### Some Other Relevant Work:

- CASL (Common Algebraic Specification Language): https://homepages.inf.ed.ac.uk/dts/pub/cai.pdf
It has equational logic with partial functions, subsorts, axioms, and a module system. But there are no tempral properties or any state predicates. 

- Iris / separation-logic frameworks: 
Iris has a rich specification language with ghost state, invariants, atomic propositions. It can specify stateful libraries. But Iris specs are written in higher-order separation logic — which is not algebraic. It is maybe too-expressive, higher-order, and very manual proofs.

- Hidden Algebra: (seems closest but haven't gone in full depth)
It is an algebraic framework for specifying stateful, object-like, black-box systems. the idea is : do not specify the concrete representation of a state; specify only what can be observed through the public interface (observers). State predicates are first-class. Two states are equivalent iff no observer sequence/experiment can distinguish them (Behavorial Equivalence - seems like bisimulation). It has no ghost variables and no decision-procedure backend. But this seems like a good specification start. It would treat a library as a hidden state transformer with a public algebraic interface.


### What all we need the specification language to have:
