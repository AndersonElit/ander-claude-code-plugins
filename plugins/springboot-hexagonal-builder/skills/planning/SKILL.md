---
name: planning
description: Guides interactive requirement gathering for development tasks. Use when starting an interactive-dev workflow to interview the user about requirements.
---

# Planning Skill

This skill guides you through conducting effective requirement interviews for development tasks. The goal is to gather enough information to create clear, testable done criteria.

## Interview Philosophy

- **Start broad, get specific**: Begin with high-level questions about the feature's purpose before diving into details
- **Provide recommendations**: Each question should have a recommended option to help users who are unsure
- **Respect user expertise**: Allow "Other (specify)" options for users who know exactly what they want
- **Know when to stop**: Stop asking when you have enough clarity to define specific done criteria

## Question Categories

### 1. Scope Questions
Define the boundaries of the feature.

Example questions:
- "What's the main goal of this feature?"
- "Which users will use this feature?"
- "What's the minimum viable version of this feature?"

### 2. Data Questions
Understand the data involved.

Example questions:
- "What information needs to be displayed?"
- "Where does this data come from?"
- "What happens if data is missing?"

### 3. UI/UX Questions
Clarify the visual and interaction design.

Example questions:
- "Where should this feature appear in the app?"
- "What should happen when the user clicks/submits?"
- "How should loading states be handled?"

### 4. Edge Case Questions
Handle unusual situations.

Example questions:
- "What happens with empty data?"
- "What error messages should appear?"
- "How should the feature behave offline?"

### 5. Technical Constraint Questions
Identify technical requirements and limitations.

Example questions:
- "Are there performance requirements?"
- "Does this need to work on specific browsers/devices?"
- "Are there accessibility requirements?"

## Question Structure

Each question should have:

1. **Clear question text**: Specific and unambiguous
2. **3-5 options**: Including a recommended choice (marked with "(Recommended)")
3. **Option descriptions**: Brief explanation of each choice's implications
4. **multiSelect flag**: Set to true when multiple options can apply

### Example Question Format

```
Question: "How should form validation work?"
Header: "Validation"
Options:
  - label: "Real-time validation (Recommended)"
    description: "Validate as user types, show errors immediately"
  - label: "On submit only"
    description: "Validate when form is submitted"
  - label: "On blur"
    description: "Validate when user leaves each field"
  - label: "No validation"
    description: "Accept any input"
```

## Interview Flow

### Phase 1: Understanding (1-2 questions)
- What is the user trying to build?
- What problem does it solve?

### Phase 2: Scope Definition (2-3 questions)
- What are the core features?
- What's out of scope for now?

### Phase 3: Details (2-4 questions)
- Specific UI/UX decisions
- Data handling
- Error cases

### Phase 4: Technical (1-2 questions, if needed)
- Performance requirements
- Browser/device support

## When to Stop Asking

Stop the interview when you can answer "yes" to all of these:

1. **Clear purpose**: You understand why this feature exists
2. **Defined scope**: You know what's in and out of scope
3. **Testable outcomes**: You can describe specific, observable behaviors
4. **Edge cases covered**: You know how errors and empty states should behave

## Output Format

After the interview, summarize requirements in this format:

```markdown
## Requirements
- [Core requirement 1]
- [Core requirement 2]
- [UI requirement]
- [Data requirement]
- [Error handling requirement]

## Technical Decisions
- [Framework/library choices]
- [Architecture decisions]
- [Browser/device support]
```

## Tips for Effective Interviews

1. **Don't over-ask**: 5-8 questions is usually enough
2. **Group related questions**: Use multiSelect to combine related choices
3. **Provide context**: Explain why you're asking if the question might seem odd
4. **Accept defaults**: If the user keeps choosing recommended options, they likely want a standard implementation
5. **Summarize understanding**: Before defining done criteria, confirm your understanding of the requirements
