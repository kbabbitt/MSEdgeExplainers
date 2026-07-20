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
    - [The `parse()` function](#the-parse-function)
    - [Scenario 1: Flowing in text](#scenario-1-flowing-in-text)
    - [Scenario 2: Typing indicator](#scenario-2-typing-indicator)
    - [Scenario 3: Loading shimmer](#scenario-3-loading-shimmer)
    - [Details and open questions](#details-and-open-questions)
      - [Nested elements](#nested-elements)
      - [Words spanning element boundaries](#words-spanning-element-boundaries)
      - [Generated content](#generated-content)
  - [Alternatives considered](#alternatives-considered)
    - [Text animation properties](#text-animation-properties)
    - [Custom Highlights](#custom-highlights)
    - [`word-index()` instead of using counters](#word-index-instead-of-using-counters)
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

- Provide a means of styling and animating text at sub-element units.

### Non-goals

- **Defining how sub-element units are determined across languages.**
Different languages have widely varying rules for how text units can be grouped into lines
of text, word units, or even letter units. The idea of addressing sub-element use of text depends
on these concepts, but we do not seek to define them here.
Implementations already need to work with these concepts in order to perform text layout, and
CSS defers the exact details of these breaks to implementations; see
[CSS Text 3](https://drafts.csswg.org/css-text-3/#line-breaking).
Additionally, [Intl.Segmenter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/Segmenter)
provides one set of breaking algorithms available to author script; an implementation that
supports Intl.Segmenter could reuse those algorithms to implement this proposal.

## User research

https://github.com/w3c/csswg-drafts/issues/3208


## Proposed Approach

### `::nth-word()` and `::nth-letter()` pseudo-elements

### The `parse()` function

https://github.com/w3c/csswg-drafts/issues/12488

<!--
### Dependencies on non-stable features

[If your proposed solution depends on any other features that haven't been either implemented by
multiple browser engines or adopted by a standards working group (that is, not just a W3C community
group), list them here.]

[[No such dependencies.]]
-->

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

### Details and open questions

#### Nested elements

Text unit counting only counts words within the target element,
not descendants. This gives authors flexibility to treat certain
sub-elements as atomic units rather than counting the words in them.
For example:

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
  p::nth-word(n),
  p > ::is(b, i, u)::nth-word(n) {
    counter-increment: wordcount;
  }
  p > ::is(pre, table, img) {
    /* treat these as atomic units */
    counter-increment: wordcount;
  }
```

#### Words spanning element boundaries

#### Generated content

`::nth-word()` and `::nth-letter()` should work with any generated content pseudo-elements
that are themselves tree-abiding.

```css
.new-item::before {
  content: "New Item";
}
.new-item::before::nth-word(2n) {
  color: red;
}
.new-item::before::nth-word(2n+1) {
  color: blue;
}
```

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

- Highlight pseudo-elements currently do not support opacity, transition, or animation properties.
- It's not practical to run the interpolation in JS because there's no inline style on highlight-affected text. You would need one ::highlight() rule per possible opacity value in a given frame. Would also give up composition.
- If we define CSS transition behavior for highlight ranges, we would still need to block-ify text and maintain style state at sub-element levels. Seems like a similar degree of complexity to ::nth-word().
- The author would need to drive movement of the Range through Javascript instead of having a purely declarative solution.
- As a consequence, staggered-start animations would be constrained by main-thread frame budget and refresh interval.
Some designs call for stagger intervals below the 16.7ms refresh interval on a 60Hz monitor.

### `word-index()` instead of using counters

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

Thanks to the following proposals, projects, libraries, frameworks, and
languages for their work on similar problems that influenced this proposal.

- [GSAP SplitText](https://gsap.com/docs/v3/Plugins/SplitText/)
- [SVG 2 Text](https://svgwg.org/svg2-draft/text.html)
