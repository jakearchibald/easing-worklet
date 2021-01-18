This is an proposal to allow developers to extend the set of [easing functions](https://drafts.csswg.org/css-easing/) available in a document.

# The current state of things

[CSS Easing Functions Level 1](https://drafts.csswg.org/css-easing/) defines a set of easing methods, along with [`cubic-bezier()`](https://drafts.csswg.org/css-easing/#funcdef-cubic-bezier-easing-function-cubic-bezier) which allows a limited degree of customisation.

```css
@keyframes fly-in {
  from {
    transform: translateY(-400%);
    animation-timing-function: ease-in-out;
  }
}

.element:hover {
  opacity: 0.5;
  transition: opacity 600px cubic-bezier(0.76, 0, 0.24, 1);
}
```

```js
element.animate(
  {
    opacity: 0.5,
  },
  {
    duration: 600,
    easing: 'ease-in',
  },
);
```

These easing functions are currently used in [CSS animations](https://drafts.csswg.org/css-animations/#animation-timing-function), [CSS transitions](https://drafts.csswg.org/css-transitions/#transition-timing-function-property), and the [web animations API](https://drafts.csswg.org/web-animations-1/#dictdef-effecttiming), although in future they may be used in non-temporal uses, like painting gradients.

Developers generally copy-and-paste or import `cubic-bezier` easings from a [predefined set](https://easings.net/), or use [custom-tooling](https://cubic-bezier.com/#.17,.67,.83,.67) to edit the curve. Few developers manually write cubic-bezier easing definitions, and few can quickly read a cubic-bezier easing definition and picture the result.

`cubic-bezier` is limited to two points, meaning you can't use it to define more complex easing, especially if the easing has multiple 'phases', such as 'bounce' or 'spring'. To work around this, developers either end up using an [animation library](https://animejs.com/documentation/#springPhysicsEasing), which prevents browser optimisations like compositor-driven animations, or [try to recreate the effect with keyframes](https://easings.net/#easeOutBounce), which is often unconvincing.

# The proposal

The idea is to allow developers to define their own easing functions with JavaScript, as part of a worklet.

## Defining an easing function

From a document:

```js
CSS.easingWorklet.addModule('custom-easings.js');
```

And in `custom-easings.js`:

```js
registerEasing(
  'elastic',
  class ElasticEasing {
    static get inputArguments() {
      return ['<number>', '<number>'];
    }

    constructor([amplitude = 1, frequency = 10 / 3]) {
      this.period = 1 / frequency;
      this.amplitude = amplitude;
    }

    ease(position) {
      // position is 0-1.
      // This function should return a number where 0 is the start position, and 1 is the end.
      // Returning position as-is would be a linear easing.
      // Here's a rough 'elastic' definition as an example:
      if (position == 0 || position == 1) return position;
      const s =
        this.amplitude < 1
          ? this.period / 4
          : (this.period / (2 * Math.PI)) * Math.asin(1 / this.amplitude);
      return (
        this.amplitude *
          Math.pow(2, -10 * position) *
          Math.sin(((position - s) * (2 * Math.PI)) / this.period) +
        1
      );
    }
  },
);
```

This easing is registered for the current document only.

This API is heavily influenced by the design of [paint worklets](https://drafts.css-houdini.org/css-paint-api-1/) and [animation worklets](https://drafts.css-houdini.org/css-animation-worklet-1/).

## Using a custom easing function

```css
@keyframes move-in {
  from {
    transform: translateY(-400%);
    animation-timing-function: ease(elastic, 1.2, 5);
  }
}
```

The CSS `ease` function can be used anywhere easing can be used. It takes the name of the easing function (as passed to `registerEasing`), and the remaining parameters are input arguments, as defined by `inputArguments`. These are passed to the constructor of the easing class.

If the input arguments are invalid, or no easing is defined for that name, an invalid value is returned. If an easing of that name is later defined, then the style is reevaluated, similar to how CSS behaves when a custom property is updated.

## How values are calculated

When values from a custom easing are needed, an instance of the class is created (although an existing instance may be used if the input arguments match). Then, `ease` is called on that instance, with the [input progress value](https://drafts.csswg.org/css-easing/#input-progress-value).

The result is expected to be deterministic, and browsers may cache results for a given progress value and input arguments and reuse it later. The browser may also generate values up-front. Eg if the browser is about to run a 5 second animation, it may generate many seconds worth of easing values up-front.

# Q&A

Here are some questions I anticipate. However, none of the answers are set in stone.

## Why not define easings in CSS?

There's a [separate proposal for CSS-defined easings](https://github.com/w3c/csswg-drafts/issues/229). This proposal isn't meant to replace that, although this proposal may be a quicker way to get the feature to developers, since it builds on existing technologies like JavaScript and worklets, and can be debugged with existing browser tooling.

## Should this be a main thread API

A main thread API would be a lot simpler for developers to use, but I don't think this should be done on the main thread.

Currently, browsers can run some animations off the main thread. If the easing functions were defined on the main thread, the easing values would need to be generated up-front, or the animation would need to yield back to the main thread for easing values.

Up-front generation could create a noticeable moment of jank at the start of the animation, especially for lengthy animations.

Also, it's unclear what the impact would be on non-animation use of easings. Eg, if easings are used for gradients, it can mean yielding back to the main thread during scrolling to paint part of the page.

## Should this be done with an animation worklet instead?

Animation worklets give developers control over the whole animation and can be used to create things like [spring animations](https://drafts.css-houdini.org/css-animation-worklet-1/#example-1), but if you're just wanting a custom easing, they're a sledgehammer to crack a nut.

Animation worklet animations need to be instantiated with JS, whereas easing worklets let you use custom easings everywhere they're currently used in the platform. This includes future non-animation use of easings, such as gradients.

## Should this be part of the animation worklet?

Technically they could share a worklet global, but since easings aren't strictly for animation, it seems like they should be different APIs.

## Why use a constructor?

The API could be:

```js
registerEasing('elastic', {
  inputArguments: ['<number>', '<number>'],
  ease(position, inputArgs) {
    // …
  },
});
```

But I used a class to be consistent with paint & animation worklets. A constructor allows for up-front calculations that don't need to be done per call, but I don't have strong feelings.
