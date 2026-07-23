# CSS Text Transitions & Animations

## Author

- [Kevin Babbitt](https://github.com/kbabbitt) (Microsoft)

## Participate

- [Issue tracker](https://github.com/MicrosoftEdge/MSEdgeExplainers/labels/CSSTextTransitions)
- [Open a new issue](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/new?labels=CSSTextTransitions&title=%5BCSSTextTransitions%5D+)

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [CSS Text Transitions \& Animations](#css-text-transitions--animations)
  - [Author](#author)
  - [Participate](#participate)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [User-Facing Problem](#user-facing-problem)
    - [Goals](#goals)
    - [Non-goals](#non-goals)
  - [User research](#user-research)
  - [Proposed Approach](#proposed-approach)
    - [`::nth-word()` and `::nth-letter()` pseudo-elements](#nth-word-and-nth-letter-pseudo-elements)
      - [Tree-abiding versus non-tree-abiding](#tree-abiding-versus-non-tree-abiding)
    - [The `parse()` function](#the-parse-function)
    - [Scenario 1: Flowing in text](#scenario-1-flowing-in-text)
      - [Variant: Flowing in text with block descendants](#variant-flowing-in-text-with-block-descendants)
    - [Scenario 2: Typing indicator](#scenario-2-typing-indicator)
    - [Scenario 3: Loading shimmer](#scenario-3-loading-shimmer)
  - [Alternatives considered](#alternatives-considered)
    - [Text animation properties](#text-animation-properties)
    - [Custom Highlights](#custom-highlights)
  - [Prior Art](#prior-art)
  - [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
    - [Accessibility](#accessibility)
    - [Internationalization](#internationalization)
    - [Privacy and Security](#privacy-and-security)
  - [References \& acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

This explainer proposes new CSS features to allow for effects to be applied
to units of text (such as words) within a given element.

## User-Facing Problem

Many Web experiences animate text at sub-element granularity. Examples include:

- AI chat interfaces that apply staggered fade-ins to each successive word, so
  that response text flows in smoothly at a steady rate.
- Typing indicators that display "..." with each dot animating in sequence.
- Loading or placeholder text that shimmers across words or characters to
  indicate progress.

One challenge with such effects is that the unit of currency for animations on
the Web is the element. Effects such as those described above require authors to
split each word (or character) into its own element, such as a `<span>`, and
apply effects individually to each such element. Doing so introduces several
problems:

- **Performance**: Each additional element adds cost to the DOM, style
  calculation, and layout, compared to having simple paragraphs of text.
- **Accessibility**: Screen readers may not correctly announce text that has
  been fragmented into many `<span>` elements, potentially reading individual
  fragments rather than flowing sentences.
- **Editing interaction**: Text selection and copy-and-paste behavior can be
  adversely affected.

Additionally, it puts the requirement on Web authors to perform the text
splitting. JavaScript `string.split()` can work when the desired unit is the
word, and packages such as
[GSAP SplitText](https://gsap.com/docs/v3/Plugins/SplitText/) do exist to
stagger animations on character, word, or line units. But the browser engine
needs to do these things anyway to perform layout, so there's an opportunity to
reuse that logic for animation purposes.

### Goals

- Provide a means of styling and animating text at sub-element units: words and letters.

### Non-goals

- **Defining how sub-element units are determined across languages.**
Different languages have widely varying rules for how text units can be grouped into
"word" or "letter" units. The idea of addressing sub-element use of text depends
on these concepts, but we do not seek to define them here.
Historically, CSS has deferred the exact details of these breaks to implementations and/or
other standards; see [CSS Text 3](https://drafts.csswg.org/css-text-3) which defers to
[UAX29](https://www.unicode.org/reports/tr29/tr29-47.html).

## User research

https://github.com/w3c/csswg-drafts/issues/3208


## Proposed Approach

### `::nth-word()` and `::nth-letter()` pseudo-elements

These pseudo-elements which select, respectively, the nth word or
grapheme cluster ("letter") in the text associated with an element.
Both selectors accept "an+b" style arguments, similar to `::nth-child()`.

Word and letter counting include units in inline descendants. This is necessary to maintain
a coherent notion of "word" in cases where inline descendant boundaries fall in the middle
of words.

Word and letter counting skip over block descendants. This gives authors flexibility
in determining how block descendants participate in stagger effects. An author might
choose to treat a `<pre>` containing a code sample as a single unit, for example, but
treat each cell in a `<table>` as a word-like unit.

#### Tree-abiding versus non-tree-abiding

A major decision point is whether or not these pseudo-elements are
[tree-abiding](https://drafts.csswg.org/css-pseudo-4/#treelike).
Unlike `::first-line`, `::nth-word()` and `::nth-letter()` can be made tree-abiding
because determining each selected region of text is not dependent on layout.

The key advantage of making these pseudo-elements tree-abiding is that it
allows for the use of CSS counters, which gives authors greater flexibility in
per-word styling and integrating effects with surrounding content.
However, making words tree-abiding also presents a complication:
how to handle element boundaries that appear in the middle of words.

Another approach would be to make these pseudo-elements non-tree-abiding,
much like `::highlight()`, which would sidestep element overlap issues but trade
away counter support. If we did that, staggering effects could still be achieved
through introduction of `word-index()` and `letter-index()` functions, akin to
`sibling-index()`. However, this approach is much less flexible than counters.
It trades away a great deal of author control over things like how block descendants
integrate with per-word flow, and how per-word effects are coordinated across
block boxes (sibling paragraphs, for example).

### The `parse()` function

https://github.com/w3c/csswg-drafts/issues/12488

Hypothetically, tree-abiding `::nth-word()` and `::nth-letter()`,
used in conjunction with CSS counters,
would unlock a powerful capability: parameterizing CSS property values as a function of text position.

However, there is one functional gap: CSS counters are rendered as strings rather than numeric values,
so they cannot be used directly in `calc()` expressions to compute things like animation delay or color channel values.
The proposed `parse()` function bridges this gap by letting the author parse the counter string to a numeric value.

*Footnote: Tomas Rezac found a way to bridge this gap today with clever use of custom property registrations;
see [blog post](https://dev.to/rezi/css-counting-magic-converting-counter-values-to-variables-e9m)
and [Codepen](https://codepen.io/rezi-the-lessful/pen/gbpNPaR).
It would be much nicer to have this supported directly by the platform, though.*

### Scenario 1: Flowing in text

Authors could achieve a flow-in animation as follows:

```html
<style>
  .fade-in-text {
    counter-reset: wordcount;
  }
  .fade-in-text::nth-word(n) {
    opacity: 1;
    transition: opacity 600ms;
    transition-delay: calc(6ms * parse(<number> counter(wordcount)));
  }
  @starting-style {
    .fade-in-text::nth-word(n) {
      opacity: 0;
    }
  }
</style>
```

![Flowing in text animation](images/text-stream.gif)

#### Variant: Flowing in text with block descendants

This example demonstrates the power of having access to counters for these scenarios.

```html
<p>
  Here is the <b>code sample</b> you asked for:
  <pre>
    int main() {
      printf("Hello world\n");
    }
  </pre>
  Compile it with your favorite <b>C compiler</b> and run it.
</p>
```

```css
  p {
    counter-reset: wordcount;
  }
  p::nth-word(n) {
    counter-increment: wordcount;
  }
  /* treat these as atomic units */
  p > ::is(pre, img) {
    counter-increment: wordcount;
  }
  /* stagger in tables cell-by-cell */
  p td {
    counter-increment: wordcount;
  }
```

### Scenario 2: Typing indicator

Authors could animate "..." dots bouncing up and down in sequence:

```html
<style>
  @keyframes dot-bounce {
    0%,
    50%,
    100% {
      transform: translateY(0);
    }
    25% {
      transform: translateY(-0.3em);
    }
  }
  .typing-dots {
    counter-reset: lettercount;
  }
  .typing-dots::nth-letter(n) {
    animation: dot-bounce 1s ease-in-out infinite;
    animation-delay: calc(150ms * parse(<number> counter(lettercount)));
    counter-increment: lettercount;
  }
</style>
<!-- ... -->
<div class="typing-dots">...</div>
```

![Typing indicator animation](images/dot-bounce.gif)

### Scenario 3: Loading shimmer

Authors could apply a looping shimmer effect that fades across characters:

```html
<style>
  @keyframes shimmer {
    0%,
    100% {
      color: #999;
    }
    50% {
      color: #333;
    }
  }
  .loading-text {
    counter-reset: lettercount;
  }
  .loading-text::nth-letter(n) {
    animation: shimmer 1.5s ease-in-out infinite;
    animation-delay: calc(100ms * parse(<number> counter(lettercount)));
    counter-increment: lettercount;
  }
</style>
<!-- ... -->
<p class="loading-text">Loading...</p>
```

![Loading shimmer animation](images/loading-shimmer.gif)

## Alternatives considered

### Text animation properties

A previous verstion of this explainer contemplated four new CSS properties:

```
transition-text-interval: <time [0s,∞]>#
transition-text-unit: [ none | character | word | line ]#
animation-text-interval: <time [0s,∞]>#
animation-text-unit: [ none | character | word | line ]#
```

**`*-text-unit`** specifies the unit of text that the transition or animation is
applied to progressively. When set to a value other than `none`, each successive
unit within the element starts its transition or animation after a staggered
delay.

**`*-text-interval`** specifies the delay between successive text units
beginning their transition or animation. For example, if
`transition-text-interval` is `6ms`, `transition-text-unit` is `word`, and
`transition-duration` is `100ms`, the first word starts immediately, the second
word starts at 6 ms, the third at 12 ms, and so on. The first word subsequently
finishes at 100 ms, the second at 106 ms, and so on.

Issues with this approach:

* Too tightly focused on animation scenarios.
* Loses representability of intermediate values.

### Custom Highlights

One might contemplate using the
[CSS Custom Highlight API](https://developer.mozilla.org/en-US/docs/Web/API/CSS_Custom_Highlight_API)
to solve the "flow-in text" scenario.
However, there are a few issues with this approach:

* Highlight pseudo-elements currently do not support opacity, transition, or animation properties.
* Supporting transitions as text enters or exits a highlight range would still require sub-element text
  fragmentation work similar to what we need for ::nth-letter() and ::nth-word().
* Authors would still need to do the work to determine "word" boundaries in JavaScript.
* Authors would also need to drive movement of Ranges in JavaScript.
  As a consequence, staggered-start animations would be constrained by main-thread frame budget and refresh interval.
* It's not practical to interpolate opacity in JavaScript because there's no inline style on highlight-affected text.

## Prior Art

SVG has long supported rich text animation capabilities, including the ability
to animate individual characters along paths and apply per-glyph transformations
(see [SVG 2 Text](https://svgwg.org/svg2-draft/text.html)). Notably, SVG can
rotate individual characters but cannot fade them independently. While SVG's
text capabilities serve as useful precedent, they do not directly address the
needs of HTML/CSS content. Extending support to SVG text elements could be
explored in the future but is out of scope for this initial proposal.

## Accessibility, Internationalization, Privacy, and Security Considerations

### Accessibility

This feature can improve accessibility over current practice. Today, authors who
want to animate text at sub-element granularity must split text into many
`<span>` elements, which can interfere with screen reader announcement and
copy-paste behavior. By allowing the browser to handle per-unit animation
natively, the DOM remains clean and semantically meaningful.

On some platforms, users may express preferences for reduced animation effects.
In CSS, this preference may be exposed via the `prefers-reduced-motion` media
feature. Authors can use this media feature to adjust their animation effects
accordingly.

### Internationalization

The order in which text units animate should follow the writing mode. In a
left-to-right context, units animate left-to-right; in a right-to-left context,
they animate right-to-left. For vertical writing modes, the order follows the
block and inline flow direction accordingly.

### Privacy and Security

No privacy or security implications have been reported against this feature.

<!--
## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

[[TBD.]]
-->

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Daniel Clark
- Hoch Hochkeppel
- Kurt Catti-Schmidt
- Mike Jackson
- Sushanth Rajasankar
- Tab Atkins Jr.

Thanks to the following proposals, projects, libraries, frameworks, and
languages for their work on similar problems that influenced this proposal.

- [GSAP SplitText](https://gsap.com/docs/v3/Plugins/SplitText/)
- [SVG 2 Text](https://svgwg.org/svg2-draft/text.html)
