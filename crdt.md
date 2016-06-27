# What Are CRDTs and why do they matter?

## EC for availablity

## Amazon Dynamo Shopping Cart

## Principled

"I was making CRDTs before they were called CRDTs"

## Re-usable

## Not too much kool-aid

Not a panacea, solve only one problem well right now "no more manual
merge function" but only if your data model suits the existing types
and semantics.

### Invariants

EC is still EC

### Performance/efficiency

Deltas
Serialisation
O(n) for merge, is there a better way?

### Size

How do you decompose?

### New Types

Making CRDTs is hard is why we have CRDTs, init?

### Composition
Map? LASP? And??

### Action-At-A-Distance

Not every client can be a replica all the time

#### Idempotence

This is unsolved unless every client is a replica or you have strong channels, see "exactly once transfer" etc

## The future
?

