---
description: "Coding style and clean code: immutability, file organization, naming, DRY, encapsulation, error handling, validation"
alwaysApply: true
---

# Coding Style & Clean Code

## Immutability (CRITICAL)

ALWAYS create new objects, NEVER mutate existing ones:

```
// Pseudocode
WRONG:  modify(original, field, value) → changes original in-place
CORRECT: update(original, field, value) → returns new copy with change
```

Rationale: Immutable data prevents hidden side effects, makes debugging easier, and enables safe concurrency.

## Constants Over Magic Numbers

- Replace hard-coded values with named constants
- Use descriptive constant names that explain the value's purpose
- Keep constants at the top of the file or in a dedicated constants file

## Meaningful Names

- Variables, functions, and classes should reveal their purpose
- Names should explain why something exists and how it's used
- Avoid abbreviations unless they're universally understood

## Smart Comments

- Don't comment on what the code does; make the code self-documenting
- Use comments to explain why something is done a certain way
- Document APIs, complex algorithms, and non-obvious side effects

## Single Responsibility

- Each function should do exactly one thing
- Functions should be small and focused (under 50 lines)
- If a function needs a comment to explain what it does, split it

## DRY (Don't Repeat Yourself)

- Extract repeated code into reusable functions
- Share common logic through proper abstraction
- Maintain single sources of truth

## File Organization & Structure

MANY SMALL FILES over FEW LARGE FILES:

- High cohesion, low coupling
- 200-400 lines typical, 800 max
- Extract utilities from large modules
- Organize by feature/domain, not by type
- Keep related code together in a logical hierarchy
- Use consistent file and folder naming conventions

## Encapsulation

- Hide implementation details
- Expose clear interfaces
- Move nested conditionals into well-named functions

## Error Handling

ALWAYS handle errors comprehensively:

- Handle errors explicitly at every level
- Provide user-friendly error messages in UI-facing code
- Log detailed error context on the server side
- Never silently swallow errors

## Input Validation

ALWAYS validate at system boundaries:

- Validate all user input before processing
- Use schema-based validation where available
- Fail fast with clear error messages
- Never trust external data (API responses, user input, file content)

## Code Quality Maintenance

- Refactor continuously
- Fix technical debt early
- Leave code cleaner than you found it

## Testing (Code-Level)

- Write tests before fixing bugs
- Keep tests readable and maintainable
- Test edge cases and error conditions

(For full TDD workflow and coverage, see the testing rule.)

## Version Control (Code-Level)

- Write clear commit messages (see git workflow rule for format)
- Make small, focused commits
- Use meaningful branch names

## Code Quality Checklist

Before marking work complete:

- Code is readable and well-named
- Functions are small and single-purpose (under 50 lines)
- Files are focused (under 800 lines)
- No deep nesting (max 4 levels)
- Proper error handling
- No hardcoded values (use constants or config)
- No mutation (immutable patterns used)
- No duplicated logic (DRY)
- Clear interfaces; implementation details hidden