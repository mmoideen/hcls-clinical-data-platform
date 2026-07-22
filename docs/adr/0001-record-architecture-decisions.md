# 0001 — Record architecture decisions

**Status:** Accepted

## Context

Architecture choices get made once and questioned many times. Six months after a decision, the people asking "why is it like this" are usually not the people who decided, and the original constraints are no longer visible in the artefact. What follows is either relitigation of a settled question, or — worse — a silent reversal by someone who assumed the current shape was accidental.

This is especially acute in a regulated environment, where a reviewer's first question about any control is not "what does it do" but "why is it sufficient". A template answers the first question. Only a written decision answers the second.

## Decision

Architecture decisions are recorded as numbered Markdown files in `docs/adr/`, using the format Michael Nygard described in *Documenting Architecture Decisions* (2011).

Each record states the context that forced a choice, the decision, and the consequences — including the bad ones. Records also state **what would change the decision**, because a decision without a reversal condition is a preference.

Accepted records are immutable. A decision that no longer holds gets a new record that supersedes the old one; the old file stays, marked superseded. Editing history to look consistent destroys the thing that makes the history useful.

## Consequences

Writing a record costs perhaps twenty minutes. Not writing one costs an hour of archaeology every time the question resurfaces, and occasionally costs a correct decision that gets undone by someone who could not see the reasoning.

The set is deliberately small. An ADR per resource would be noise; these cover the choices where a competent engineer could reasonably have gone the other way.
