# JSON-LD Verifiable Credentials: Predicates vs Type Properties

This repo demonstrates a common confusion (which we also had when working on
the 0.6 UNTP data models) when object modeling with JSON-LD and RDF: a **predicate**
does not need to be a **type property**.

To demonstrate the confusion, we're using here the simpler models of credentials
(without proofs for brevity) which Steve proposed in his email exchange with
Manu - Person and Address, with two separate credentials:

1. A [`person-address.jsonld`](./person-address.jsonld) credential with a Person as the credential subject,
   and relating the person with the address where they reside, and
2. An [`address-census.jsonld`](./address-census.jsonld) credential with an Address as the credential
   subject, and relating the address to the residents that live at that address.

## The Confusion

When attempting to model data using JSON-LD, with two types defined in
[`types-context.jsonld`](./types-context.jsonld) for `Person` and `Address`, and
wanting to model where a `Person` lives, we would hit the following error:

```json
{
  "name": "jsonld.ValidationError",
  "details": {
    "event": {
      "type": [ "JsonLdEvent" ],
      "code": "invalid property",
      "level": "warning",
      "message": "Dropping property that did not expand into an absolute IRI or keyword.",
      "details": { "property": "residesAt", "expandedProperty": "residesAt" }
    }
  }
}
```

The most obvious but false assumption to a developer less familiar with JSON-LD
and RDF, is that the `residesAt` property must be defined on the type being used
as the credentialSubject - similar to object-oriented programming where classes declare
their properties. This lead us to thinking we needed to either:

1. Add `residesAt` as a property of the `Person` type (not great - it's not part of the `Person`), or
2. Create hybrid type like `PersonAddress` which either extends or encapsulates a `Person` to include an `Address`.

Either of these solutions "fixes" the error but complicate the model by actually modelling relationships via the model types - **this is incorrect in RDF/JSON-LD.**

## The Reality: Predicates are Independent from Type Properties

In RDF, **predicates** (relationships) exist independently of types and type properties. They define relationships between entities, not properties that "belong to" types.

### What needs IRI definitions:

- **Types**: `Person`, `Address` (what things are)
- **Predicates**: `residesAt`, `hasResidents` (relationships between things)
- **NOT**: Types don't need to pre-declare which predicates can be used with them (we were implicitly trying to do this because we mistook the predicate for a type property in the error message).

## Testing the Concept

### See the Error
1. Remove the `residesAt` predicate definition from the context of [`person-address.jsonld`](./person-address.jsonld), so it contains only the vc and local context files:
   ```json
   {
     "@context": [
       "https://www.w3.org/ns/credentials/v2",
       "./types-context.jsonld"
     ]
   }
   ```

2. Run: `npm run jsonld`
3. Observe the "dropping property" error for `residesAt`

```json
{
  "name": "jsonld.ValidationError",
  "details": {
    "event": {
      "type": [ "JsonLdEvent" ],
      "code": "invalid property",
      "level": "warning",
      "message": "Dropping property that did not expand into an absolute IRI or keyword.",
      "details": { "property": "residesAt", "expandedProperty": "residesAt" }
    }
  }
}
```

### See the Fix
1. Add back the predicate definition:
   ```json
   {
     "@context": [
       "https://www.w3.org/ns/credentials/v2",
       "./types-context.jsonld",
       {
         "residesAt": "http://example.org/residesAt"
       }
     ]
   }
   ```

2. Run: `npm run jsonld`
3. Validation succeeds **without** having to change or add properties to the type.

## Key Insights

1. **Predicates are global relationships** - they can connect any entities that make semantic sense
2. **Types define entity structure** - what properties an entity inherently has
3. **No circular dependencies needed** - `Person` doesn't need to know about `residesAt`, and `Address` doesn't need to know about `hasResidents`
4. **Flexibility** - new relationships can be added without modifying existing type definitions
5. **Reusability** - the same types can be used in different contexts with different predicates

## Implication for UNTP credentials

We can go back to having a `Product` as the subject of the DPP, rather than a
`ProductClaim` composition of product and claims, while still representing the
desired relationships and remaining valid jsonld. v0.5 of UNTP had invalid
JSON-LD because we were redefining types, while trying to avoid that, we ended
up seeing the above errors about missing properties and so went down the track
of defining composite models. We can undo that with 0.7, assuming we can add the
predicates as in this example, using Jargon or otherwise with post-processing of
Jargon output (or some other strategy).

TODO:
- [ ] Create issue / improvement PR for jsonld-cli to avoid devs making wrong assumptions
- [ ] Verify we can add predicates to contexts in Jargon or work with Alistair to find a solution there.

## Commands

- `npm run jsonld` - Validate and expand both credentials
- Individual validation: `./node_modules/.bin/jsonld expand --safe --lint <file>`
