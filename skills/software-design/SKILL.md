---
name: software-design
description: Design principles from "A Philosophy of Software Design" (Ousterhout). Use when designing a new feature, module, interface, or package layout, or when reviewing/refactoring for complexity. Gives a checklist, red flags, and decision rules.
---

# Software Design Checklist (Ousterhout)

**One goal: minimize complexity.** Complexity = anything that makes code hard to understand or modify.
Complexity grows step by step: many small "just this once" decisions add up, so care about small things.

## Symptoms (how to recognize it)
- **Change amplification** – a small change needs edits in many places.
- **Cognitive load** – you must remember a lot to make a change safely.
- **Unknown unknowns** – you can't tell what must change or what will break. (Worst one.)

## Causes
- **Dependencies** – code can't be understood or changed alone; it needs other code.
- **Obscurity** – important information is hidden or unclear.

## Mindset
- **Strategic, not tactical.** Working code is not enough. Spend ~10–20% extra effort on design in every change. No quick hacks ("tactical tornado" = a developer who writes fast but leaves a mess); the bad design costs more and more over time.
- **Design it twice.** Sketch 2+ very different designs for any important interface and compare them. The first idea is rarely the best.
- **Fix design problems when you touch the code** – leave it a bit better than you found it.
- **Design for ease of reading, not writing.** The readers decide if code is obvious, not the author. Complexity is easier to see for readers than for writers. Complexity in a place that is rarely touched is almost as good as no complexity. More lines can be simpler; number of lines is not a measure of complexity.
- If you aren't making the design better, you're probably making it worse. Ask: is this the best I can do within the limits I have?
- Clean design and speed work together; simpler code is often faster too. Don't ignore speed in basic choices, but measure before and after optimizing, and undo changes that don't help (unless they make code simpler). On the hot path (code that runs most often), handle special cases with one early check.

## 1. Deep modules (the core idea)
- Module = class, function, package, service. Value = functionality hidden ÷ interface complexity.
- **Deep**: simple interface, lots of functionality behind it (Unix file I/O, GC). **Shallow**: interface about as complex as implementation (red flag).
- **Prefer simple interface over simple implementation.** Keep the complexity inside the module; there are more users than authors.
- **Make the common case trivial.** Good defaults, "do the right thing" without being asked. Rare features must not make life harder for common users (no forced `Buffered(File(...))` chains).
- Don't split into many small classes/methods just because they're "too long". Length alone is not a reason. Split only if it gives cleaner abstractions. "Classitis" = too many small classes, so more interfaces and more complexity.
- Depth matters more than length. Long methods are OK when they have a simple interface and read cleanly. Each method: do one thing, completely.

## 2. Information hiding & leakage
- Each module owns a few design decisions (formats, protocols, algorithms, data structures) and does not show them in its interface.
- **Leakage** = the same knowledge in several modules (e.g. reader and writer both know the file format). Hidden leakage (not visible in the interface) is the worst.
- Fix leakage by: merging the modules, or moving the shared knowledge into a new module with a simple interface.
- `private` is not information hiding if getters/setters give access to the internals anyway.
- **Avoid temporal decomposition** – don't split modules by the order things happen (read → parse → write). Split by *knowledge*.
- Making a class a little *bigger* often hides more.
- Don't hide what callers really need (e.g. performance settings they must tune).

## 3. General-purpose, "somewhat"
- Build only what today needs, but make the interface general (`insert(pos, text)`, `delete(start, end)` instead of `backspace()`, `deleteSelection()`).
- General-purpose interfaces are usually simpler, deeper, and need less code.
- Questions: What is the simplest interface that covers all current needs? In how many situations will this method be used? Is it easy to use for today's needs?
- **Move specific code up** (UI/feature code) **or down** (drivers/plugins behind a generic interface). Keep the middle general. Don't mix special-purpose code into a general mechanism.
- **Remove special cases**: design the normal case so it also covers the edge cases (an empty selection instead of a `hasSelection` flag + if-checks).

## 4. Layers & abstractions
- **Different layer, different abstraction.** If two layers that call each other (one directly above the other) do the same kind of work, that is a red flag.
- **Pass-through methods** (only call another method with the same parameters): the work is split in the wrong way. Let callers use the lower class directly, move work between the classes, or merge them.
- Same signature in several places is fine for dispatchers and for many implementations of one interface (adapters, drivers).
- **Decorators/wrappers** (classes that wrap another class to add behavior) are often shallow. First try: put the behavior in the underlying class, merge it with the code that uses it, merge it with an existing decorator, or make it standalone.
- The interface should look different from the internal data (lines stored inside, character-based API outside).
- **Pass-through variables** (passed through layers that don't use them): use a shared object or a **context object** (one object that carries shared values; immutable if possible). Avoid globals.

## 5. Pull complexity downwards
- When complexity can't be avoided, handle it inside the module, not by pushing it to callers.
- **Configuration parameters are usually an easy way out that makes life harder for users.** Compute the value automatically (e.g. adaptive retry timeout) or give a good default. Expose it only if users can really choose better than the module can.
- Don't overdo: pull complexity down only if it belongs to the module's main job, makes callers simpler, and makes its interface simpler. Otherwise it's just leakage.

## 6. Together or apart?
**Combine when:** they share information; they are always used together; they are about the same idea; it is hard to understand one without the other; combining makes the interface simpler or removes duplicate code.
**Separate when:** one is general-purpose and the other special-purpose.
- Splitting adds interfaces, distance (jumping between files), and duplicate code.
- **Conjoined methods** (you can't understand one without reading the other) = bad split.
- Don't make a separate class/method for a logging line or another tiny one-use piece.

## 7. Define errors out of existence
- Exceptions/error branches add a lot of complexity (many callers must handle them).
- Techniques, in order of preference:
  1. **Define the error away** – change the meaning so the situation is not an error (deleting a key that doesn't exist = it is already gone; substring with out-of-range indexes returns the part that overlaps).
  2. **Mask** – handle it at a low level so callers never see it (retry inside TCP).
  3. **Aggregate** – one handler at a high level for many failures.
  4. **Crash** – for errors that can't reasonably be handled (out of memory).
- Why: error-handling code is hard to test and often fails when needed (most catastrophic failures come from wrong error handling). Throwing is easy, handling is hard; classes with many exceptions have shallow, complex interfaces.
- Masking works best at a low level; aggregation works best when errors travel up several levels to one handler.
- Fewer places that handle an error = better. Special cases are also complexity.
- Crash caveats: don't abort on I/O errors in replicated storage; don't turn frequent errors into fatal ones.
- Don't overdo: errors that matter to callers must stay visible.

## 8. Comments and documentation
- Comments say what code can't: **intent, reason (why), the big idea, constraints that are not obvious.** Write them for a new reader, not for yourself.
- **Comments are a design tool.** Write interface comments *first*. If the comment is hard to write or long, the interface is bad.
- Interface comment: what a caller needs to know (behavior, units, ranges, special cases, side effects, conditions that must be true before calling). **Not** how it is implemented (red flag: implementation details leaking into the interface comment).
- "Good code is self-documenting" is a myth: comments carry the informal interface and the reasons. Every class and method needs an interface comment.
- Implementation comments: explain what the code does and *why* where it is not obvious. Put comments before big blocks, tricky logic, and bug fixes. Put each comment as close as possible to the code it describes; high-level comments are easier to keep up to date.
- Test: could someone who never saw the code write this comment just by looking at the code next to it? If yes, the comment adds nothing.
- Put the rationale of a change in the code, not only in the commit message.
- Don't repeat the code in words ("increment i"). If a reader can guess it from the name, skip it.
- High-level and low-level comments: high-level ones give the general idea, detailed ones give exact facts (units, limits, null, who owns the data, rules that must always be true).
- **Decisions that affect several modules**: write them in one central place (e.g. a designNotes file or the most relevant type) and add short links from the code.
- Don't copy documentation, including external docs. Link to the other place.
- Don't argue "it's obvious": if a reviewer says it's not, it isn't.
- Keep docs near the code; check the diff for needed doc updates before committing.
- Write comments first, then code (costs about 5% of dev time): comments written afterwards are worse and often skipped.

## 9. Naming
- A name gives the reader a clear picture of what the thing is (and isn't).
- **Precise**: `numActiveIndexlets` not `count`; `cursorVisible` not `blinkStatus`; `NOT_YET_VOTED` not `SENTINEL`. Boolean names should read like a yes/no question (`isEmpty`, `cursorVisible`). No `result` in functions that don't return it. Not too specific either (`selection` arg on a generic `delete`).
- **Consistent**: same name for the same concept everywhere; never use the name for anything else; one name = one behavior. (One `block` name used for two meanings caused a real bug.) Use different types where possible (`FileBlock` vs `DiskBlock`).
- **Remove extra words** (`fileObject`, `FileBlock` inside class `File`, Hungarian prefixes = type letters in the name like `strName`).
- Distance: short names (`i`, `n`) are fine in a few lines; the farther the use is from the declaration, the longer the name.
- **Hard to name = warning sign for bad design.** The thing probably has an unclear purpose or does several jobs. Redesign.

## 10. Consistency & obviousness
- Consistency = same conventions for naming, patterns, interfaces, and rules that always hold (invariants). Write them down, enforce them with linters, and follow them in new code. "When in Rome, do as the Romans do": use the existing pattern unless you have a much better reason (and then change all of them).
- **Obvious code**: a reader's first guess is right. Made unclear by: generic containers (`Pair<Pair<…>>`, `map[string]any`), event-driven/dispatch indirection that hides the control flow, code that does not do what readers expect, hidden side effects.
- Prefer clear types and specific names over bare tuples/maps. Comments help where code can't.
- Use invariants (rules that always hold) to keep code consistent. Don't change a convention without strong new information; don't force very different things into one pattern.
- Help the reader: blank lines separate blocks; the declared type matches the real type; code that does something unexpected (e.g. a thread that keeps running after its caller ends) needs a comment.

## 11. Decide what matters
- Find the few things that matter and make them easy to see (naming, interface docs, a central place). Hide everything else.
- **Reduce what matters**: fewer parameters, good defaults, hidden details, auto-computed settings.
- Two mistakes: treating too much as important (shallow classes, clutter) and missing something important (critical information that is hidden).
- Look for **leverage**: one general idea or rule that explains or solves many things.

## 12. Software trends (judge each by "does it reduce complexity?")
- **Agile/incremental**: yes, but each step should add an *abstraction*, not just a feature. Don't ship design debt.
- **TDD** (write tests before code): tests focus on features, not design. Don't let it replace thinking about abstractions. Writing a failing test first is good for bug fixes.
- **Inheritance**: inheriting implementation creates dependencies between parent and child. Prefer composition (one object holds another); inheriting only an interface is fine. Think hard before subclassing.
- **Design patterns**: use them when they fit; don't force them. More patterns ≠ better design.
- **Getters/setters**: shallow, and they expose internals. Avoid them; put behavior on the type.
- **Tests** make refactoring safe; without them developers avoid it. Unit and system tests are both valuable.
- **Interfaces/abstraction layers** are worth it only if they hide something.

## 13. Modifying existing code
- Stay strategic: after each change the design should look as if it was made that way from the start.
- Don't work around a bad abstraction. Refactor it, even if only a little.
- If you change behavior, update the comments in the same commit (and any places that refer to them).
- Prefer a small redesign over adding more special cases.

---

## Red flags (quick scan)

| Red flag                                      | What it means                                                                                            |
|-----------------------------------------------|----------------------------------------------------------------------------------------------------------|
| **Shallow module**                            | Interface costs as much as it gives. Usually merge or deepen it; sometimes unavoidable but still useful. |
| **Information leakage**                       | Same knowledge in 2+ places. Merge, or move it to one place.                                             |
| **Temporal decomposition**                    | Structure follows the order of steps, not knowledge.                                                     |
| **Overexposure**                              | The common case forces users to learn rare features.                                                     |
| **Pass-through method / variable**            | Layers do the same kind of work.                                                                         |
| **Repetition**                                | The same code appears again and again: an abstraction is missing.                                        |
| **Special-general mixture**                   | General code contains code for one specific use case.                                                    |
| **Conjoined methods**                         | Must read both to understand either. Bad split.                                                          |
| **Comment repeats code**                      | Adds nothing. Delete it or write at a higher level.                                                      |
| **Implementation docs contaminate interface** | Interface comment explains how it works, not what it does.                                               |
| **Vague name**                                | Name could mean many things → people misuse it.                                                          |
| **Hard to pick a name**                       | The design is not clear.                                                                                 |
| **Hard to describe**                          | Interface comment is long or complicated → the interface is wrong.                                       |
| **Nonobvious code**                           | A reader can't guess the behavior quickly.                                                               |
| **Many special cases / exceptions**           | The normal case is not general enough; define errors away.                                               |

---

## Pre-implementation checklist for a new feature or module

1. What *knowledge/decision* does this module own? Is it owned in only one place?
2. Write the interface (signatures + comments) **first**. Can I describe it in a few clear sentences?
3. Design it **twice** with a really different alternative. Which has the simpler interface for the common case?
4. Is it deep? What complexity does it hide from callers? What does it push onto them (parameters, errors, call order, config)? Pull it down where reasonable.
5. Is it somewhat general (not named after today's UI/use case)? Is the specific code moved up or down?
6. Does each layer do a different kind of work? Any pass-through methods, wrappers, or variables passed through many layers?
7. Which errors/special cases can be defined away, masked, or aggregated? Which really must reach the caller?
8. Names: precise, consistent with existing words in the code, no extra words? Any hard-to-name thing → redesign.
9. Does it follow existing conventions and patterns in the codebase? Anything surprising for a new reader?
10. Comments: interface docs written, the *why* explained where not obvious, rules that affect several modules written in one place.
11. Dependencies: could I change the internals without touching callers? What breaks if X changes?
12. Am I thinking long-term (strategic)? Did I leave the surrounding design a bit cleaner?

## Pre-merge review questions
- Would a new reader's first guess about each name and method be right?
- Any new shallow class/method, pass-through, getter/setter, config option, or special case that a better abstraction could remove?
- Did I split something only because it was "long"?
- Does the change copy a decision into a second module?
- Do comments explain *why/what*, and are they updated?
