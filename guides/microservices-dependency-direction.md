# Dependency Direction Between Modules and Microservices

What to do when two modules or services need something from each other, why calls should go one way only, and how to tell good design from a patch.

The ideas apply the same way to modules inside one binary (a modular monolith) and to separate microservices. The examples use two contexts: **Orders** (a buyer pays for an order) and **Transfers** (money is paid out to the seller).

**Prerequisites:** none. A working Go example is in `~/code/modular-monolith`. Start with `internal/orders/internal/adapters/transfers_inprocess.go` and `internal/transfers/module.go`.

## 1. Pick one direction

- Dependencies run one way: `Orders → Transfers`. Transfers never calls Orders.
- **The consumer owns the interface.** Orders declares what it needs in its own words (`ViewPayout(orderID) → Payout`). An adapter on the Orders side translates that into the other side's public API.
- **The provider exposes a public contract, not its internals.** Transfers offers `TransferByReference(reference) → TransferView` plus its own public errors. `TransferView` is a copy, never the domain object.
- **Each side has its own types.** Orders' `Payout` is not Transfers' `TransferView`. Statuses are mapped in one place (`completed` → `paid_out`), so a rename upstream breaks one function.
- **The provider stays ignorant of its consumers.** Transfers knows an opaque `Reference` (`order:<id>`), not orders. Only the adapter knows what the reference means.
- **The direction is a choice.** It's fine for both sides to care about the same fact. What you avoid is calls in both directions.

In Go, a module's public surface can be its top-level package, with everything else under a nested `internal/`. The compiler then rejects an import of another module's internals, and it rejects import cycles between packages.

## 2. Why cycles (A ↔ B) are bad design

The formal names are **cyclic dependency** and the **Acyclic Dependencies Principle** ("allow no cycles in the component dependency graph"). "Chicken and egg" is the informal name.

What a cycle costs:

| Problem | In a monolith | Between microservices |
|---|---|---|
| **Construction order** | `NewA` needs B and `NewB` needs A, so you end up with setters and late binding | A won't become healthy until B is up, and B waits for A |
| **Deadlock** | A holds a lock while calling B, B calls back into A and waits on the same lock | A → B → A ties up threads and connections on both sides |
| **Changing and releasing** | The two modules can only change together | A contract change on either side needs a coordinated release |
| **Failure** | — | A's outage breaks B, which breaks A |
| **Size** | The cycle merges the components into one big one | It's a **distributed monolith** |

Go forbids cycles between packages, not between modules. You can make a two-way module dependency compile by wiring both adapters in the composition root. It compiles, and every problem in the table is still there.

**Distributed monolith:** services that can't be changed or deployed independently ("everything has to be deployed at the same time", Sam Newman). Calls in both directions are one common cause, not the definition. The goal is **independent deployability**.

## 3. When the upstream side needs data from the downstream side

Example: Orders already calls Transfers, and now Transfers "needs an order view". First ask **why** Transfers needs it:

| Why it's needed | Approach | Direction |
|---|---|---|
| **To do its job** (payout description, buyer country for compliance) | **Push the data in the request.** Add fields to the request and let the receiver keep its own copy. "Tell, don't ask." | Unchanged ✅ |
| **To show it** (a transfer page showing order details) | **Compose at the edge.** A composition root, BFF or API gateway calls both public APIs and merges the views. Neither side knows the other's view exists. | Neither points at the other ✅ |
| **To react to changes over time** (a cancelled order must block a pending payout) | **Events plus a projection owned by the consumer.** The upstream side publishes facts, and the consumer keeps a small read model of only the fields it needs, in its own terms, built from its own storage. | One direction ✅ |

Don't do this: Transfers declares an `OrderReader` interface and the composition root plugs Orders into it. It compiles, and it is still a cycle (see section 2).

Microsoft's guidance: if a service "needs data that's originally owned by other microservices, do not rely on making synchronous requests for that data. Instead, replicate or propagate that data (only the attributes you need) into the initial service's database by using eventual consistency."

## 4. Does async make a cycle OK?

Partly. Async removes **runtime coupling** but not **design-time coupling**.

What async fixes:
- No deadlock and no request loops holding threads.
- No cascading synchronous failure. If B is down, A keeps working and B catches up from the queue later.
- No startup ordering problem.

What async doesn't fix:
- **Contract coupling.** If A consumes B's events and B consumes A's events, a breaking change on either side still needs a coordinated release. You still have a distributed monolith, just a slower one.
- **Coupling in the business logic.** If finishing one business operation takes A → B → A → B ping-pong, the two services share one process. You've traded a deadlock for eventual-consistency problems: races, ordering, duplicate messages, harder debugging.
- **New failure mode: event loops.** A's event triggers B, B's event triggers A, and it never stops.

### Messages can flow both ways while dependencies go one way

Direction comes from **whose vocabulary each message uses**, not from which way the bytes travel:

- A **command** is written in the *receiver's* contract: "Transfers, please do a payout."
- An **event** is written in the *publisher's* contract: "Transfers: transfer X completed."

```
Orders ──RequestPayout{reference}──▶ Transfers      (Transfers' contract)
Orders ◀──TransferCompleted{reference}── Transfers  (Transfers' contract)
```

Messages go both ways, but both are in Transfers' vocabulary. Orders depends on Transfers, and Transfers knows nothing about orders. There's no cycle. ✅

```
Orders ──OrderPaid──▶ Transfers                    (Orders' contract)
Orders ◀──TransferCompleted── Transfers            (Transfers' contract)
```

Each side imports the other's events, so there's a cycle at the contract level. Making the calls async only patched it. ⚠️

**Verdict:** async is good design when it carries messages in one direction of dependency. It's a patch when the cycle is still in the contracts or in the business logic.

## 5. Checklist

1. **Can each message or call be written in exactly one side's vocabulary?** If yes, you have a direction. If no, you have a cycle.
2. **Can you deploy a new version of A while B is on an old version, or down?** If no, you have a distributed monolith, async or not.
3. **Does an exchange end after a fixed number of steps, or does it ping-pong?** Ping-pong means the boundary is in the wrong place.
4. **Do changes to A and B keep landing in the same PR or release?** Then they're one bounded context. Merge them, or move the responsibility so one side owns it. No messaging style fixes a wrong boundary.
5. **Does the consumer own the interface and its own types?** Or does it map the other side's internal structs?

## When not to use this

- **Within one bounded context.** Classes and packages inside a single module can call each other freely, as long as Go doesn't see a package cycle. Direction rules are for boundaries between contexts.
- **When the boundary itself is wrong.** If two modules always change together, merging them is cheaper than designing one-way contracts between them.
- **For throwaway prototypes.** Keep it in one module until the boundaries are clear ("monolith first").

## 6. Microservice bad smells (Taibi & Lenarduzzi, 2018)

This is a catalogue of 11 smells from interviews with 72 experienced practitioners, who reported 265 bad practices and the solutions they applied. Source: D. Taibi, V. Lenarduzzi, "On the Definition of Microservice Bad Smells", *IEEE Software* 35(3), 2018, DOI 10.1109/MS.2018.2141031. Summarised from the catalogue table in the [IEEE Computer Society article](https://www.computer.org/publications/tech-news/research/defining-bad-smells-specifically-for-cloud-native-applications-based-on-microservices).

| Smell | Description / detection | Problem it may cause | Adopted solution |
|---|---|---|---|
| **API Versioning** | APIs are not semantically versioned (no v1.1, 1.2, …). Also called Static Contract Pitfall. | After a new version, consumers may face connection issues: the returned data differs, or the API must be called differently. | Version APIs semantically, so services know whether they talk to the right version or must adapt to a new contract. |
| **Cyclic Dependency** | A cyclic chain of calls between microservices exists, e.g. A calls B, B calls C, and C calls back A. | The services in the cycle are hard to maintain or reuse in isolation. | Refine the cycles according to their shape, and apply the API Gateway pattern. |
| **ESB Usage** | Microservices communicate via an enterprise service bus. | An ESB adds complexity for registering and deregistering services. | Use a lightweight message bus instead. |
| **Hard-Coded Endpoints** | Hardcoded IP addresses and ports between connected services. Also called Hardcoded IPs and Ports. | Problems when service locations change. | Use service discovery. |
| **Inappropriate Service Intimacy** | A service keeps connecting to other services' private data instead of its own: requests for their private data, or direct connections to their databases. | More coupling. Often caused by a mistake in modelling the data. | Consider merging the services. |
| **Microservice Greedy** | New services are created for every feature even when not needed, e.g. a service that serves only one or two static HTML pages. | An explosion in the number of services, making a needlessly huge system that becomes unmaintainable. | Carefully consider whether the new service is needed. |
| **Not Having an API Gateway** | Services talk directly to each other, and in the worst case consumers talk directly to every service. | Interviewees could manage about 50 interconnected services. Beyond that they hit communication and maintenance problems. | Apply the API Gateway pattern. |
| **Shared Libraries** | Different microservices use shared libraries. | Tight coupling and lost independence. Teams must coordinate to change the library. | (1) Accept the redundancy (duplicate the code), or (2) extract the library into a separate service that can be deployed and developed independently. |
| **Shared Persistency** | Different services access the same relational database, in the worst case the same entities. Also called Data Ownership. | High coupling, less team and service independence. | (1) Independent databases per service, (2) a shared database with private tables per service, or (3) a private schema per service. |
| **Too Many Standards** | Different languages, protocols, frameworks, etc. are used. Also called the Lust and Gluttony bad practices. | Hard for the company to sustain, especially when developers leave. | Adopt different standards carefully, without following the latest hype. |
| **Wrong Cuts** | Services are split by technical layer (presentation, business, data) instead of business capability. | Wrong separation of concerns and more complex data splitting. | Analyse business processes and resource needs clearly. |

How the smells relate to this guide:
- **Cyclic Dependency** is section 2. Note that the paper's "API gateway" fix and section 3's "compose at the edge" fix are the same move.
- **Inappropriate Service Intimacy** and **Shared Persistency** are what you get when you read the other side's internals instead of its public contract, or instead of keeping your own projection.
- The paper's fix for Inappropriate Service Intimacy (merge the services) is checklist item 4.

## Sources

Principle:
- Robert C. Martin, *Clean Architecture*, ch. 14 "Component Coupling", on the Acyclic Dependencies Principle and how cycles merge components and are broken with Dependency Inversion. [O'Reilly](https://www.oreilly.com/library/view/clean-architecture-a/9780134494272/ch14.xhtml) · [Wikipedia summary](https://en.wikipedia.org/wiki/Acyclic_dependencies_principle)
- Rob Pike, "Go at Google: Language Design in the Service of Software Engineering" (2012), section 7: *"Although they are occasionally useful, circular imports introduce significant problems at scale."* [go.dev](https://go.dev/talks/2012/splash.article)
- Eric Evans, *Domain-Driven Design Reference* (2015), "Context Mapping" and "Customer/Supplier" (pp. 28–32): relationships between contexts have a direction, upstream → downstream. [PDF](https://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf)

Microservices:
- Microsoft, *.NET Microservices Architecture*, "Communication in a microservice architecture". It calls synchronous request chains an anti-pattern, says to replicate needed data via integration events, and to minimise communication between services. [Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture)
- Sam Newman, QCon London 2020, reported by InfoQ. The distributed monolith is *"the worst of the monoliths"*, and *"the goal is independent deployability."* See also his books *Building Microservices* (2nd ed.) and *Monolith to Microservices*. [InfoQ](https://www.infoq.com/news/2020/05/monolith-decomposition-newman/)
- Taibi & Lenarduzzi, "On the Definition of Microservice Bad Smells", *IEEE Software* 2018 (section 6). [IEEE Computer Society](https://www.computer.org/publications/tech-news/research/defining-bad-smells-specifically-for-cloud-native-applications-based-on-microservices)

A real chicken-and-egg incident:
- Meta, "More details about the October 4 outage" (2021). The tools needed to fix the network depended on that network: *"the total loss of DNS broke many of the internal tools we'd normally use to investigate and resolve outages like this."* [engineering.fb.com](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)

Sections 3 (the "why does it need it?" table), 4 (async and vocabulary direction) and 5 (checklist) are a synthesis, not taken from a single source.
