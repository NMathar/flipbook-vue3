Flipbook Vue 3 component

# Flipbook Vue 3

## Description

This package brings a vue 3 component that displays images in 3D page flip effect. Origin of this component is this repository: [flipbook-vue](https://github.com/ts1/flipbook-vue). Original repository provides a component that works nicely in vue 2, but due to errors and bugs in my vue 3 project I had to update it.

## Installation

To install this package, run the following command:

```bash
npm i @nmathar/flipbook-vue3
```

## Usage

You can use the `flipbook-vue3` component in your Vue 3 project as follows:

### Importing the Component

Import the component and use it in your Vue file:

```javascript
import FlipBook from '@nmathar/flipbook-vue3';
import "@nmathar/flipbook-vue3/dist/style.css"
```

Or register it as a Vue plugin:

```javascript
import { createApp } from 'vue'
import App from './App.vue'
import FlipBook from '@nmathar/flipbook-vue3';
import "@nmathar/flipbook-vue3/dist/style.css"

const app = createApp(App)
app.component('FlipBook', FlipBook)
app.mount('#app')
```


## Props

Here is a list of available props for the `flipbook-vue3` component:

### pages
Array of image URLs. Required. All images should have the same aspect ratio.
If the first element is null, the next element is displayed alone (as the cover page).
All other props are optional.

### alt
Prop that sets alt attribute for the images just for better seo performance.

### pagesHiRes
Array of high resolution versions of image URLs. They are used when zoomed.

### flipDuration
Duration of page flipping animation in milliseconds. Defaults to 1000.

### zoomDuration
Duration of zoom in/out animation in milliseconds. Defaults to 500.

### zooms
Array of possible magnifications. null is equivalent to [1] (no zoom). Defaults to [1, 2, 4]. NOTE : Do NOT pass an empty array.

### ambient
Intensity of ambient light in 0 to 1. Smaller value gives more shades. Defaults to 0.4.

### gloss
Intensity of specular light in 0 to 1. Higher value gives more gloss. Defaults to 0.6.

### perspective
Z-axis distance in pixels between the screen and the viewer. Higher value gives less effect. Defaults to 2400.

### nPolygons
How many rectangles a single page is horizontally split into. Higher value gives higher quality rendering in exchange for performance. Defaults to 10.

### singlePage
Force single page mode regardless of viewport size. Defaults to false.

### forwardDirection
Reading direction. If your document is right-to-left, set this "left". Default is "right".

### centering
Enable centering of the cover pages. Default is true.

### startPage
Page number (>= 1) to open. Default is null.

### loadingImage
URL of an image that is displayed while page is loading. By default internal animated SVG is used.

### clickToZoom
Zoom in or out on click or tap. Default is true.

### dragToFlip
Flip page by dragging/swiping. Default is true.

### wheel
When set to 'zoom', mouse wheel events zoom in/out the page. Default is 'scroll', wheel events and touch pad scroll gestures scroll the zoomed page.

## Events

The `flipbook-vue3` component supports the following events:

### flip-left-start
Fired when flip to left animation starts. Argument is page number before flip.

### flip-left-end
Fired when flip to left animation ends. Argument is page number after flip.

### flip-right-start
Fired when flip to right animation starts. Argument is page number before flip.

### flip-right-end
Fired when flip to right animation ends. Argument is page number after flip.

### zoom-start
Fired when zoom-in/out animation starts. Argument is magnification after zoom.

### zoom-end
Fired when zoom-in/out animation ends. Argument is magnification after zoom.

## Slot props

This component exposes some properties and methods as slot properties. Example usage:

```javascript
<flipbook :pages="pages" v-slot="flipbook">
  <button @click="flipbook.flipLeft">Previous Page</button>
  <button @click="flipbook.flipRight">Next Page</button>
</flipbook>
```

### canFlipLeft
True if it can flip to previous page. NOTE: Can return false if currently being animated.

### canFlipRight
True if it can flip to next page. NOTE: Can return false if currently being animated.

### canZoomIn
True if it can zoom in.

### canZoomOut
True if it can zoom out.

### page
Current page number (1 to numPages).

### numPages
Total number of pages.

### flipLeft()
Method to flip to previous page.

### flipRight()
Method to flip to next page.

### zoomIn()
Method to zoom in.

### zoomOut()
Method to zoom out.

## Usage Examples

```vue
<script setup lang="ts">
import { ref } from 'vue'
import FlipBook from '@gladesinger/flipbook-vue3';
import "@gladesinger/flipbook-vue3/dist/style.css"
import IconArrow from './components/IconArrow.vue'

// Array of image paths
const pages = ref([])
</script>

<template>
  <div class="slider-wrap">
    <FlipBook v-slot="flipbook" class="flipbook" :pages="pages" :gloss="0" :click-to-zoom="true" alt="Book slide">
      <button aria-label="Previous" class="flipbook-button button-prev" :class="{'disabled': !flipbook.canFlipLeft}" @click="flipbook.flipLeft">
        <IconArrow />
      </button>
      <button aria-label="Next" class="flipbook-button button-next" :class="{'disabled': !flipbook.canFlipRight}" @click="flipbook.flipRight">
        <IconArrow />
      </button>
    </FlipBook>
  </div>
</template>

<style scoped>
.slider-wrap {
  position: relative;
  width: 100vw;
  max-width: 1120px;
  margin: 0 auto;
}

.slider-wrap .flipbook {
  width: 100%;
  height: 410px;
  display: block;
}

.slider-wrap .flipbook img {
  max-width: 100%;
}

.slider-wrap .flipbook-button {
  position: absolute;
  top: 50%;
  cursor: pointer;
  background: none;
  border: none;
  padding: 0;
  outline: none;
}

.slider-wrap .flipbook-button.button-prev {
  left: -64px;
  transform: translateY(-50%);
}
.slider-wrap .flipbook-button.button-next {
  right: -64px;
  transform: rotate(180deg) translateY(50%);
}
.slider-wrap .flipbook-button.disabled {
  opacity: 0.35;
  cursor: auto;
  pointer-events: none;
}
</style>
```
