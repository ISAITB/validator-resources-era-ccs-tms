# Federated Semantic Interoperability for ETCS Compliance

## Validating EuroBalise Distance Rules/Compliance (wrt. SUBSET-036 §5.6.3)  Across RINF and ETCS Engineering Data

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [The Problem](#2-the-problem)
3. [The Rule: SUBSET-036 §5.6.3](#3-the-rule-subset-036-5.6.3)
4. [How It Works: The Big Picture](#4-how-it-works-the-big-picture)
5. [Data Sources and Architecture](#5-data-sources-and-architecture)
6. [Formal Specification (First-Order Logic)](#6-formal-specification-first-order-logic)
7. [SHACL Implementation](#7-shacl-implementation)
8. [Federated SPARQL Compliance Query](#8-federated-sparql-compliance-query)
9. [Results and What They Mean](#9-results-and-what-they-mean)
10. [Why This Matters](#10-why-this-matters)
11. [References](#11-references)

---

## 1. Executive Summary

European railway operations depend on data that lives in many places — the regulatory infrastructure register, the engineering design tools of each Infrastructure Manager, and the technical specifications published by the European Union Agency for Railways (ERA). Verifying that an ETCS deployment complies with the technical rules in those specifications today requires manually cross-referencing all of these sources.

This work demonstrates that the **same compliance check can be performed automatically** by federating SPARQL endpoints through the W3C standards stack — RDF, OWL, SHACL, and SPARQL — using a real ETCS rule from **SUBSET-036 §5.6.3** as the demonstration target.

The rule is straightforward to state in plain language:

> *Two consecutive Eurobalises on the same track must be a minimum centre-to-centre distance apart, where the minimum depends on the track's Maximum Permitted Speed.*

The data needed to check this rule is distributed:

- The **track's maximum permitted speed** lives in **RINF** (the ERA Register of Infrastructure).
- The **balise positions** live in an **engineering extension** maintained by the Infrastructure Manager.

Neither dataset alone can answer the compliance question. **Federated SPARQL** and **federated SHACL** can.

---

## 2. The Problem

Modern railway operation involves data from many sources:

- **ERA RINF** — the authoritative legal record of infrastructure across Europe, published as Linked Open Data via a SPARQL endpoint.
- **Infrastructure Manager engineering data** — detailed ETCS commissioning information including balise positions, marker boards, speed profiles, and switch topology.
- **TSI and SUBSET specifications** — technical rules issued by ERA defining how ETCS must be deployed.

When an engineer asks *"Are the balises on Track 1 in Vennesla station spaced correctly per ETCS System Requirements Specification?"*, today they must:

1. Look up the track's maximum permitted speed in RINF.
2. Open the engineering design tool to find balise positions.
3. Open/Understand ETCS System Requirements Specification for the rule.
4. Manually compute the distance for every balise pair.
5. Manually compare against the rule.

This is slow, error-prone, not auditable, and does not scale across the European network.

---

## 3. The Rule: SUBSET-036 §5.6.3

**Source:** ETCS System Requirements Specification, SUBSET-036, page 86.

### The Specification

> The minimum distance between two consecutive Eurobalises shall be:
>
> | Maximum Permitted Speed | Minimum distance (centre-to-centre) |
> |---|---|
> | 0 < v ≤ 180 km/h | **2.3 m** |
> | 180 < v ≤ 300 km/h | **3.0 m** |
> | 300 < v ≤ 500 km/h | **5.0 m** |

### Why It Exists

Eurobalises transmit telegrams to the train via electromagnetic induction. If two balises are too close together, their telegrams can overlap and corrupt one another — a safety risk that increases with train speed. SUBSET-036 §5.6.3 sets the minimum separation that guarantees clean telegram reception at each speed band.

### What Compliance Looks Like

For a track in Vennesla with maximum permitted speed of **80 km/h** (falling in the lowest band), every pair of balises on that track must be **at least 2.3 m apart**. A pair found at 1.8 m apart is a compliance violation that must be corrected before the track can be commissioned for ETCS service.

---

## 4. How It Works: The Big Picture

### The Three Sources of Information

```
   ┌─────────────────────┐     ┌────────────────────────┐     ┌──────────────────────┐
   │   RINF (ERA)        │     │   SUBSET-036           │     │   Extension (CCSTMS) │
   │                     │     │                        │     │                      │
   │  - Track ID         │     │  Rule §5.6.3:          │     │  - Balise positions  │
   │  - Max speed        │     │  Min distance depends  │     │  - BaliseGroups      │
   │  - Operational pts  │     │  on speed band         │     │  - ETCSMarkers, etc  │
   └─────────┬───────────┘     └──────────┬─────────────┘     └──────────┬───────────┘
             │                            │                              │
             │                            │                              │
             └──────────────┬─────────────┴───────────────┬──────────────┘
                            │                             │
                            ▼                             ▼
                   ┌─────────────────────────────────────────────┐
                   │   Federated SPARQL / SHACL Compliance Check │
                   │   1. Get track speed from RINF              │
                   │   2. Get balise positions from Extension    │
                   │   3. Apply SUBSET-036 rule encoded in logic │
                   │   4. Report compliance per balise pair      │
                   └─────────────────────────────────────────────┘
```

### The Flow

1. Start with the **station** (Vennesla) and find all its tracks via RINF's `era:hasPart`.
2. For each track, get the **maximum permitted speed** from RINF.
3. Follow the track's `era:netReference` to find its **LinearElements** — the topology pieces the track is made of.
4. Federate to the **extension endpoint** to fetch all balises positioned on each LinearElement.
5. For every pair of balises on the same LinearElement, compute the **actual centre-to-centre distance**.
6. Apply the **SUBSET-036 threshold** for the track's speed band.
7. Report each pair as `COMPLIANT` or `VIOLATION`.

No data is duplicated. No system is changed. The data stays where it lives — RINF remains the authoritative speed source, the extension remains the authoritative balise position source — and SPARQL federation does the rest.

---

## 5. Data Sources and Architecture

### The Two SPARQL Endpoints

| Role | Endpoint |
|---|---|
| Authoritative infrastructure register | `https://graph.data.era.europa.eu/repositories/rinf-plus` |
| ETCS engineering extension | `https://graphdb.praedicta.de/repositories/europes_rail` (graph `<http://norway_extended/data>`) |

### Data Model Navigation

**From RINF — finding the track speed and topology:**

```
era:OperationalPoint
  era:opName "Vennesla"@en
  era:hasPart → era:RunningTrack
                  era:maximumPermittedSpeed → integer (km/h)
                  era:netReference → era:NetLinearReference
                                       era:hasSequence → rdf:List
                                                          rdf:first → era:LinearElement
```

**From the Extension — finding the balises:**

```
era:Balise
  era:topologicalCoordinate → era:TopologicalCoordinate
                                era:onLinearElement → era:LinearElement
                                era:offsetFromOrigin → integer (mm)
```

The join key is `era:LinearElement`. The same URI appears in both datasets, which is what makes the federated join work.

---

## 6. Formal Specification (First-Order Logic)

The rule is expressed compactly in classical first-order logic. This formal representation is the canonical reference for the SHACL and SPARQL implementations that follow.

### Predicates

```
Balise(b)              — b is a Balise
LinearElement(le)      — le is a LinearElement
On(b, le)              — Balise b is placed on LinearElement le
SpeedLimit(le, s)      — The maximum permitted speed on le is s (km/h)
offset(b)              — The offsetFromOrigin of b's topologicalCoordinate (m)
```

### The Threshold Function

The minimum distance is a piecewise function of the speed:

```
                ┌  2.3   if  0   < s ≤ 180
minDist(s)  =   ┤  3.0   if  180 < s ≤ 300
                └  5.0   if  300 < s ≤ 500
```

### The Rule

```
∀ le, b1, b2, s :
    (   LinearElement(le)
     ∧  Balise(b1) ∧ Balise(b2)
     ∧  b1 ≠ b2
     ∧  On(b1, le) ∧ On(b2, le)
     ∧  SpeedLimit(le, s)
    )
    ⇒
    |offset(b1) − offset(b2)|  ≥  minDist(s)
```

In words: *for any two distinct balises on the same LinearElement, the absolute difference between their offsets must be at least the minimum required by the speed band.*

---

## 7. SHACL Implementation

The FOL rule translates directly into a SHACL shape using a `sh:SPARQLConstraint`. Because the speed lookup happens in RINF and the balise positions in the extension, the SPARQL inside the constraint uses `SERVICE` to federate.

```turtle
@prefix sh:   <http://www.w3.org/ns/shacl#> .
@prefix era:  <http://data.europa.eu/949/> .
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .

era:BaliseMinimumDistanceShape
    a sh:NodeShape ;
    rdfs:label "SUBSET-036 §5.6.3 — Minimum Balise Centre-to-Centre Distance"@en ;
    sh:targetClass era:LinearElement ;
    sh:sparql era:BaliseMinimumDistanceConstraint .

era:BaliseMinimumDistanceConstraint
    a sh:SPARQLConstraint ;
    sh:message """Balises {?b1} and {?b2} on LinearElement {$this} are
        {?actualDist_m} m apart, but SUBSET-036 §5.6.3 requires at least
        {?minDist_m} m for tracks with maximum permitted speed
        {?maxSpeed} km/h."""@en ;
    sh:prefixes era:ETCSPrefixes ;
    sh:select """
        SELECT $this ?b1 ?b2 ?actualDist_m ?minDist_m ?maxSpeed
        WHERE {
            ?b1 era:topologicalCoordinate ?tc1 .
            ?tc1 era:onLinearElement   $this ;
                 era:offsetFromOrigin  ?offset1_mm .

            ?b2 era:topologicalCoordinate ?tc2 .
            ?tc2 era:onLinearElement   $this ;
                 era:offsetFromOrigin  ?offset2_mm .

            FILTER(STR(?b1) < STR(?b2))

            SERVICE <https://graph.data.era.europa.eu/repositories/rinf-plus> {
                ?track a era:RunningTrack ;
                       era:maximumPermittedSpeed ?maxSpeed ;
                       era:netReference ?netLinearRef .
                ?netLinearRef era:hasSequence ?seqList .
                ?seqList rdf:rest*/rdf:first $this .
            }

            BIND(?offset1_mm * 0.001 AS ?offset1_m)
            BIND(?offset2_mm * 0.001 AS ?offset2_m)
            BIND(ABS(?offset1_m - ?offset2_m) AS ?actualDist_m)

            BIND(
                IF(?maxSpeed > 300 && ?maxSpeed <= 500, 5.0,
                IF(?maxSpeed > 180 && ?maxSpeed <= 300, 3.0,
                IF(?maxSpeed > 0   && ?maxSpeed <= 180, 2.3,
                                                        2.3))) AS ?minDist_m
            )

            FILTER(?actualDist_m < ?minDist_m)
        }
    """ .

era:ETCSPrefixes
    a sh:PrefixDeclaration ;
    sh:declare [ sh:prefix "era"  ; sh:namespace "http://data.europa.eu/949/"^^xsd:anyURI ] ,
               [ sh:prefix "rdf"  ; sh:namespace "http://www.w3.org/1999/02/22-rdf-syntax-ns#"^^xsd:anyURI ] ,
               [ sh:prefix "rdfs" ; sh:namespace "http://www.w3.org/2000/01/rdf-schema#"^^xsd:anyURI ] .
```

### Mapping FOL → SHACL

| FOL Element | SHACL/SPARQL Element |
|---|---|
| `∀ le : LinearElement` | `sh:targetClass era:LinearElement` |
| `∀ b1, b2 : Balise on le` | Two `era:topologicalCoordinate` patterns sharing `$this` |
| `b1 ≠ b2` | `FILTER(STR(?b1) < STR(?b2))` (also deduplicates symmetric pairs) |
| `SpeedLimit(le, s)` | `SERVICE <RINF>` block fetching `era:maximumPermittedSpeed` |
| `minDist(s)` | Nested `IF` binding `?minDist_m` |
| `|offset(b1) − offset(b2)| ≥ minDist(s)` | `FILTER(?actualDist_m < ?minDist_m)` reports the **negation** |

### Why Federated SHACL Works

The SHACL standard does not specify federation, but every major SHACL engine that implements the optional `sh:SPARQLConstraint` feature supports `SERVICE` clauses inside the SPARQL — because `sh:select` is just a SPARQL query. This works on Apache Jena, TopBraid, and RDF4J/GraphDB out of the box.

---

## 8. Federated SPARQL Compliance Query

The same logic as a direct query produces a per-pair compliance report.

```sparql
PREFIX era:  <http://data.europa.eu/949/>
PREFIX bnd:  <https://data.banenor.no/data/>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?trackLabel ?linearElement ?maxSpeed ?speedBand
       ?b1 ?b2 ?offset1_m ?offset2_m
       ?actualDist_m ?minDist_m
       ?complianceStatus ?marginOrGap_m
WHERE {
  ?station a era:OperationalPoint ;
           era:hasPart ?track ;
           era:opName "Vennesla"@en .

  ?track a era:RunningTrack ;
         rdfs:label ?trackLabel ;
         era:maximumPermittedSpeed ?maxSpeed ;
         era:netReference ?netLinearRef .

  FILTER(LANG(?trackLabel) = "en")

  ?netLinearRef era:hasSequence ?sequenceList .
  ?sequenceList rdf:rest*/rdf:first ?linearElement .

  SERVICE <https://graphdb.praedicta.de/repositories/europes_rail> {
    GRAPH <http://norway_extended/data> {
      ?b1 era:topologicalCoordinate ?tc1 .
      ?tc1 era:onLinearElement   ?linearElement ;
           era:offsetFromOrigin  ?offset1_mm .

      ?b2 era:topologicalCoordinate ?tc2 .
      ?tc2 era:onLinearElement   ?linearElement ;
           era:offsetFromOrigin  ?offset2_mm .

      FILTER(STR(?b1) < STR(?b2))
    }
  }

  BIND(?offset1_mm * 0.001 AS ?offset1_m)
  BIND(?offset2_mm * 0.001 AS ?offset2_m)
  BIND(ABS(?offset1_m - ?offset2_m) AS ?actualDist_m)

  BIND(
    IF(?maxSpeed > 300 && ?maxSpeed <= 500, 5.0,
    IF(?maxSpeed > 180 && ?maxSpeed <= 300, 3.0,
    IF(?maxSpeed > 0   && ?maxSpeed <= 180, 2.3,
                                            2.3))) AS ?minDist_m
  )

  BIND(
    IF(?maxSpeed > 300 && ?maxSpeed <= 500, "BAND_C: 300<v≤500 km/h",
    IF(?maxSpeed > 180 && ?maxSpeed <= 300, "BAND_B: 180<v≤300 km/h",
    IF(?maxSpeed > 0   && ?maxSpeed <= 180, "BAND_A: 0<v≤180 km/h",
                                            "OUT_OF_RANGE"))) AS ?speedBand
  )

  BIND(IF(?actualDist_m >= ?minDist_m, "COMPLIANT", "VIOLATION")
       AS ?complianceStatus)

  BIND(?actualDist_m - ?minDist_m AS ?marginOrGap_m)
}
ORDER BY ?complianceStatus ?marginOrGap_m ?trackLabel ?linearElement
```

### Summary Variant — Per-Track Compliance Roll-Up

For dashboards and executive reporting:

```sparql
PREFIX era:  <http://data.europa.eu/949/>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?trackLabel ?maxSpeed ?minDist_m
       (COUNT(*) AS ?totalPairs)
       (SUM(IF(?actualDist_m >= ?minDist_m, 1, 0)) AS ?compliantPairs)
       (SUM(IF(?actualDist_m <  ?minDist_m, 1, 0)) AS ?violations)
WHERE {
  ?station a era:OperationalPoint ;
           era:hasPart ?track ;
           era:opName "Vennesla"@en .

  ?track a era:RunningTrack ;
         rdfs:label ?trackLabel ;
         era:maximumPermittedSpeed ?maxSpeed ;
         era:netReference ?netLinearRef .

  FILTER(LANG(?trackLabel) = "en")

  ?netLinearRef era:hasSequence ?sequenceList .
  ?sequenceList rdf:rest*/rdf:first ?linearElement .

  SERVICE <https://graphdb.praedicta.de/repositories/europes_rail> {
    GRAPH <http://norway_extended/data> {
      ?b1 era:topologicalCoordinate ?tc1 .
      ?tc1 era:onLinearElement ?linearElement ;
           era:offsetFromOrigin ?offset1_mm .
      ?b2 era:topologicalCoordinate ?tc2 .
      ?tc2 era:onLinearElement ?linearElement ;
           era:offsetFromOrigin ?offset2_mm .
      FILTER(STR(?b1) < STR(?b2))
    }
  }

  BIND(ABS(?offset1_mm - ?offset2_mm) * 0.001 AS ?actualDist_m)
  BIND(
    IF(?maxSpeed > 300 && ?maxSpeed <= 500, 5.0,
    IF(?maxSpeed > 180 && ?maxSpeed <= 300, 3.0,
                                            2.3)) AS ?minDist_m
  )
}
GROUP BY ?trackLabel ?maxSpeed ?minDist_m
ORDER BY ?trackLabel
```

---

## 9. Results and What They Mean

### A Sample Compliance Report Looks Like:

| trackLabel | maxSpeed | speedBand | b1 (short) | b2 (short) | actualDist_m | minDist_m | complianceStatus | marginOrGap_m |
|---|---|---|---|---|---|---|---|---|
| 1 | 80 | BAND_A: 0<v≤180 km/h | balise/abc-1 | balise/def-2 | 5.42 | 2.3 | COMPLIANT | +3.12 |
| 2 | 80 | BAND_A: 0<v≤180 km/h | balise/ghi-3 | balise/jkl-4 | 2.85 | 2.3 | COMPLIANT | +0.55 |

### Interpreting the Margins

- **Large positive margin** — comfortable compliance, no concern.
- **Small positive margin** — compliant but close to the limit; worth flagging for design review.
- **Negative margin** — direct violation of SUBSET-036; must be corrected before commissioning.

### What an Auditor Sees

Each row of the result is **traceable** to its sources:
- The speed limit traces to a specific RINF triple.
- The balise positions trace to specific extension triples.
- The threshold traces to SUBSET-036 §5.6.3 via the SHACL shape's `rdfs:comment`.

This is the audit trail that manual cross-referencing cannot provide.

---

## 10. Why This Matters

### For Infrastructure Managers

- **Automated commissioning checks** — what once took hours of manual cross-referencing now runs as a single query.
- **Reduced commissioning risk** — catch SUBSET-036 violations before the track goes live, not after.
- **Repeatable verification** — the same shape and query run identically across every station, line, and corridor.

### For National Safety Authorities and ERA

- **Traceable, auditable compliance** — every result links back to specific data triples and rule clauses.
- **Standards-based verification** — built entirely on W3C RDF, SHACL, and SPARQL — no proprietary tooling.
- **Cross-border consistency** — the same rule applies identically whether the data is from Norway, Germany, or Spain.

### For the European Interoperability Agenda

This demonstrates that the ERA ontology is not just a publication format — it is a **semantic interoperability fabric** capable of supporting safety-critical reasoning across organisational and system boundaries. The data stays where it lives, ownership stays with the authoritative source, and **compliance becomes a query**.

### The Pattern Is Generic

SUBSET-036 §5.6.3 is just one rule. The same pattern — federated SHACL + SPARQL across RINF and engineering extensions — applies to others (more sample below):

- Contact wire height vs TSI ENE Table 4.2.9.1
- Switch branch speed discontinuity checks
- ETCS marker placement vs gradient
- BaliseGroup coverage vs braking distance at switches
- Balise Installation in narrow curves

Each is a federated query waiting to be written. The infrastructure to do so already exists.

---

## 11. References

- **ETCS SUBSET-036** — Eurobalise FFFIS, §5.6.3, page 86.
- **EU Regulation 2016/797** — Interoperability of the rail system within the European Union.
- **TSI CCS** — Technical Specification for Interoperability for Control-Command and Signalling.
- **ERA Ontology** — [https://data-interop.era.europa.eu/](https://data-interop.era.europa.eu/)
- **Extended ERA Ontology** — [CCSTMS extension](https://gitlab.com/era-europa-eu/public/interoperable-data-programme/era-ontology/era-ontology/-/tree/ext-ccstms?ref_type=heads)
- **RINF SPARQL endpoint** — [https://graph.data.era.europa.eu/repositories/rinf-plus](https://graph.data.era.europa.eu/repositories/rinf-plus)
- **W3C SHACL** — [https://www.w3.org/TR/shacl/](https://www.w3.org/TR/shacl/)
- **W3C SPARQL 1.1 Federation** — [https://www.w3.org/TR/sparql11-federated-query/](https://www.w3.org/TR/sparql11-federated-query/)

---

*Document prepared as part of a federated semantic interoperability demonstration for ETCS engineering compliance.*