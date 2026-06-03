# Algebraic specification of stateful libraries with Trace-based Reasoning backends

Stateful library specifications can have two kinds of views: a trace view (what HAT uses — finite sequences of events with temporal logic over them) and an algebraic view (constructors, observers, equational laws). HAT's machinery makes the trace view the implementation (decidable inclusion via SFA), but the algebraic view is what humans want to write and reason about. Stateful libraries are naturally specified algebraically — as equational theories of constructors and observers, with invariants as state predicates.  

For example: A developer thinking about a Set thinks in equations:
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

Algebraic systems such as KAT and KMT provide nice and clean equational reasoning, but they do not directly target symbolic trace specifications of black-box stateful libraries with first-class ghost variables and event predicates. KAT/KMT provide an algebraic style of reasoning, where programs/specifications are terms and verification can be phrased as equations or inclusions. That style is much nicer: specs can be thought and reasoned about equationally. KMT can encode client theories, but then ghost state and symbolic predicates tend to be pushed into the theory, rather than being first-class trace/specification constructs. They have the right "algebraic frontend, decidable backend" pattern but at the wrong granularity (programs, not data types). We want the usability and compositionality of algebraic specifications, but the semantic target should remain symbolic traces of black-box library interfaces.

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

### Translation:

The translation obligation is: from algebraic laws and state predicates, emit a symbolic-trace property — with the right event guards and ghost variables — inside a fragment where SFA inclusion is decidable, and sound with respect to the observational meaning of the equations.

### Some Other Relevant Work:

- CASL (Common Algebraic Specification Language): https://homepages.inf.ed.ac.uk/dts/pub/cai.pdf

It has equational logic with partial functions, subsorts, axioms, and a module system. But there are no tempral properties or any state predicates. 

- Iris / separation-logic frameworks: 

Iris has a rich specification language with ghost state, invariants, atomic propositions. It can specify stateful libraries. But Iris specs are written in higher-order separation logic — which is not algebraic. It is maybe too-expressive, higher-order, and very manual proofs.

- Hidden Algebra: 

It is an algebraic framework for specifying stateful, object-like, black-box systems. the idea is : do not specify the concrete representation of a state; specify only what can be observed through the public interface (observers). State predicates are first-class. Two states are equivalent iff no observer sequence/experiment can distinguish them (Behavorial Equivalence - seems like bisimulation). It has no ghost variables and no decision-procedure backend. But this seems like a good specification start. It would treat a library as a hidden state transformer with a public algebraic interface.


### Problem statement:

We want to come up with an algebraic specification language for stateful library interfaces whose terms are equational and compositional, whose primitives include first-class state predicates and ghost state, and whose semantics is a translation into symbolic traces such that spec entailment = SFA language inclusion.


## Running examples:

### An example from HAT - Set on Key-Value Store:

#### Library Interface, Algebraically: (KVStore)

Signatures:
```
empty  : Store  (or state)                   -- constructor
put    : Store x Key x Val -> Store          -- constructor
exists : Store x Key -> Bool                 -- observer
get    : Store x Key -> Val                  -- observer 
```
Axioms:
```
exists(empty, k)                  = false
exists(put(s,k,v), k)             = true
k != k'  =>  exists(put(s,k,v), k') = exists(s, k')
get(put(s,k,v), k) = v 
k != k'  =>  get(put(s,k,v), k')    = get(s, k')
```
-- derived: distinct keys commute, same key overwrites
k != k'  =>  put(put(s,k,v), k',v') = put(put(s,k',v'), k, v)
(else)      put(put(s,k,v), k, v') = put(s, k, v')

#### Clients, as equations over the library: (Set)

```
empty  : Set                          -- constructor
insert : Set × Elem → Set             -- constructor
mem    : Set × Elem → Bool            -- observer
```
```
mem(insert(s,x), x) = true
x != y  =>  mem(insert(s,x), y) = mem(s, y)
mem(empty, x)                = false
insert(insert(s,x), x) = insert(s,x)     
```
The mem lines are observer equations. The line insert(insert(s,x),x) = insert(s,x) is a behavioural equation between constructor terms: it asserts two states are observationally indistinguishable. It is not a trace property to check but a coinductive consequence of the observer equations. mem (the observer) cannot separate the two states.

HAT spec:
```
mem(x)          =  ∃ k. exists(s,k) ∧ get(s,k) = x          -- scan; elided
insert_guard(x) =  if mem(x) then () else put(freshKey(), x)
insert_idem(x)  =  put(key(x), x)        where key : Elem ↪ Key   (injective)
```

#### Invariant: 

The ideal specification for the representation invariant: (what user writes)
```
Inv(s) = ∀ k1 k2.  exists(s,k1) ∧ exists(s,k2) ∧ get(s,k1) = get(s,k2)  =>  k1 = k2    -- a predicate over state
```
With ghost parameters:
```
Inv(s, el) = ∀ k1 k2.  exists(s,k1) ∧ exists(s,k2) ∧ get(s,k1) = el ∧ get(s,k2) = el  =>  k1 = k2
Inv(s) = ∀el. Inv(s,el)
```
Equivalently, per value, at most one witness(key) (writing Inv as a bound on Set), i.e. for compilation to SFA:
```
W(s, el) = { k | exists(s,k) ∧ get(s,k) = el }  -- set of witness
Inv(s)   = ∀ el.  |W(s, el)| ≤ 1
```
If we fix the ghost variable el:
```
W(put(s,k,v), el) =
   W(s,el) ∪ {k}     if v = el
   W(s,el) \ {k}     if v != el ∧ k ∈ W(s,el)
   W(s,el)           otherwise
```
Now we have three (relevant) states |W| = 0 | 1 | ≥ 2 

Note that these states we reason about are observational constructed from predicates and not based on the library's concrete internals:

RI in HAT:
```
I_Set(el)  =  □ ( ⟨put k v = ν | v = el⟩ ⟹ ○ □ ¬⟨put k' v' = ν | v' = el⟩ )
```

Inv is a "better" RI than I_Set since it alllows something like:
```
t1 = ⟨put k a⟩ . ⟨put k a⟩       -— same key, re-put 
t2 = ⟨put k1 a⟩ . ⟨put k1 b⟩ . ⟨put k2 a⟩        -— k1 overwritten: 
```

#### Challenges:

- Going from state predicate to trace language: The translation must (a) know how each event changes the state and read off the axioms, so it can speak of "the state a trace reaches"; and (b) convert the predicate's single yes/no into "holds at every prefix,".

- Ghost synthesis: The developer writes no ghost; el appears only after translation. The translation must manufacture the ghost and decide what to universally quantify the emitted property over. Also, choosing which ghost to have so that you don't go outside the decidable fragment.

- The axioms/equations only tell what changes, and we have to derive/infer what doesn't change and turn it into operational content. Like W(put(s,k,v), el) = W(s,el) \ {k} if v != el ∧ k ∈ W(s,el) is derived from axiom get(put(s,k,v),k)=v.

- How to reduce the client specification: If we try reducing every method to traces it produces to check them, this is just symbolic execution of the client.


### An example from Hidden Algebra to avoid the circularity of the HAT benchmarks:

```
th FLAG is sort Flag .
pr DATA .
ops (up_) (dn_) (rev_) : Flag -> Flag .
op up?_ : Flag -> Bool .
var F : Flag .
eq up? up F = true .
eq up? dn F = false .
eq up? rev F = ¬ up? F .
endth
```

A flag is raised by up, lowered by dn, flipped by rev, observed by up?.

We want to verify that rev (rev F) behaves like F — reversing twice restores the flag - proved by coinduction

the up? value is a parity condition, and this is exactly the property LTL_f cannot express. So we are forced to introduce an explicit mod-2 ghost counter going through every event and assert an invariant about it. But this cannot be an algebraic spec and needs to be an operational transition system. This ghost var is a different kind of ghost than last example and we need to identify which type we need in any example. Note that we are still in the regular expression world.

The ghost state bit : Bool updates as follows:
```
⟨up⟩      :  bit = true       -- from  up?(up F)  = true
⟨dn⟩      :  bit = false      -- from  up?(dn F)  = false
⟨rev⟩     :  bit = ¬ bit      -- from  up?(rev F) = ¬ up?(F)
⟨up? = ν⟩ :  ν = bit    -- the observation reports the counter
```

### Challenges:

- The above ghost stuff to handle "up? rev F = ¬ up? F". This is the problematic part since the value it writes depends on what's already there (The observer is a function of it's previous value), and this type of pattern is non trivial to cover in translation. - worst case we assume no "observer-transforming" transitions

- Need to make sure the algebra only denote regular languages (since regular algebra including the one in Hidden Algebra can express more) - or come up with an extention to the decision procedure(s).

- The equality of two states here is Behavioural equality, ie observer cannot tell them apart, instead of something like "they were built the same way". But the backend is trace-based, which forces the property over histories(traces), which is closer to the second part(in quotes). An example of this would be from prev example in I_Set with ⟨put k1 a⟩.⟨put k1 a⟩ vs ⟨put k1 a⟩. Thus the final trace property must depend only on reached state even though it's written over event histories.