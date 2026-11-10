---
name: ux-writing
description: >-
  Interface copy rules: voice, button and link labels, errors, empty states, settings labels, placeholders. Use when writing or reviewing the words in a product interface. Not for how text renders ([[typography]]), error markup ([[accessibility]]), prose meant to rank in search ([[seo]]) or whole-screen audits ([[interface-review]]).
---

# Writing that disappears into the interface

The project's own instructions win: where its AGENTS.md, CLAUDE.md, design system or established conventions say otherwise, follow them and skip that part of this. Use the project's existing kit and tokens; do not add components, on-screen text or options that the project's design does not call for.

Fewer words is the default. Do not add a sentence that explains a screen; if a screen needs explaining, raise that as a design problem. These rules are for the words that do have to be there.

**Start from the existing voice.** Read nearby copy, the product's terms and any style guide before writing. The product has one voice and one name for each thing: if the menu says "Archive", the toast does not say "Move to storage". A difference from generic plain language is a problem only when it causes inconsistency, ambiguity, translation trouble or the wrong tone for the stakes.

**Tone follows the stakes.** Routine actions and settings are neutral and minimal. Errors, destructive confirmations, data loss and security are calm and plain, with no playfulness, no "oops" and no exclamation marks.

**Plain, direct wording.** Address the reader as "you". Prefer a direct status to a "we" that deflects: "Unable to load content", not "We're having trouble loading this content". Use possessives sparingly ("Favorites", not "Your Favorites"). Match the device verb: "tap" on touch, "click" with a pointer, "select" when both apply. Avoid idioms and humor that will not translate.

**Buttons start with the verb for the specific action.** "Save draft", "Delete project". A confirmation's buttons repeat the consequence so the dialog can be answered without reading its body: `Delete project` and `Cancel`, not `Yes` and `No`. A multi-step flow uses one word to advance throughout.

**Links name their destination.** Link text makes sense out of context, because screen-reader users navigate by the list of links. No "Click here", and no bare "Learn more" repeated on a page.

**One capitalization policy per element type.** Sentence case is the default: it is calmer and localizes cleanly.

**A toggle is labelled for its on state.** "Send read receipts", not "Don't send read receipts". Link to a setting instead of describing the path to it.

**Empty and error states follow one standard pattern.** Say what is missing or what went wrong, and give the one action that fixes it. Keep it as short as it can be, with no paragraph explaining the screen. An empty list is a short line and its action ("No projects yet" with `Create project`). An empty search or filter names the query and offers the way out. An error is an instruction placed beside what failed: "Choose a password with at least 8 characters", not "That password is too short"; "Unable to save. Check your connection and try again.", not a generic failure. Do not blame the reader. Use the same wording pattern on every view, in whichever of the project's standard empty and error variants fits the content's shape ([[layout]]); do not write a one-off state for a single screen. If the same error keeps firing, the interaction needs redesigning, not rewording.

**A placeholder is an example of the format**, never the field's only label, because it disappears on input.

**Strings are whole templates.** Do not build sentences by joining fragments around variables; word order changes by language. Use full templated strings with proper pluralization.

When reviewing copy, quote the current text and give the complete replacement, not a direction to move in.
