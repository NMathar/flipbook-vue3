<template>
  <div>
    <slot v-bind="{
      canFlipLeft,
      canFlipRight,
      canZoomIn,
      canZoomOut,
      page,
      numPages,
      flipLeft,
      flipRight,
      zoomIn,
      zoomOut,
    }" />
    <div ref="viewport" class="viewport" :class="{
      zoom: zooming || zoom > 1,
      'drag-to-scroll': dragToScroll,
    }" :style="{ cursor: cursor == 'grabbing' ? 'grabbing' : 'auto' }" @touchmove="onTouchMove"
      @pointermove="onPointerMove" @mousemove="onMouseMove" @touchend="onTouchEnd" @touchcancel="onTouchEnd"
      @pointerup="onPointerUp" @pointercancel="onPointerUp" @mouseup="onMouseUp" @wheel="onWheel">
      <div class="flipbook-container" :style="{ transform: `scale(${zoom})` }">
        <div class="click-to-flip left" :style="{ cursor: canFlipLeft ? 'pointer' : 'auto' }" @click="flipLeft" />
        <div class="click-to-flip right" :style="{ cursor: canFlipRight ? 'pointer' : 'auto' }" @click="flipRight" />
        <div :style="{ transform: `translateX(${centerOffsetSmoothed}px)` }">
          <img v-if="showLeftPage" class="page fixed" :style="{
            width: pageWidth + 'px',
            height: pageHeight + 'px',
            left: xMargin + 'px',
            top: yMargin + 'px',
          }" :src="pageUrlLoading(leftPage, true)" @load="didLoadImage($event)" :alt="alt" />
          <img v-if="showRightPage" class="page fixed" :style="{
            width: pageWidth + 'px',
            height: pageHeight + 'px',
            left: viewWidth / 2 + 'px',
            top: yMargin + 'px',
          }" :src="pageUrlLoading(rightPage, true)" @load="didLoadImage($event)" :alt="alt" />

          <div :style="{ opacity: flip.opacity }">
            <div v-for="[
              key,
              bgImage,
              lighting,
              bgPos,
              transform,
              z,
            ] in polygonArray" :key="key" class="polygon" :class="{ blank: !bgImage }" :style="{
                backgroundImage: bgImage && `url(${loadImage(bgImage)})`,
                backgroundSize: polygonBgSize,
                backgroundPosition: bgPos,
                width: polygonWidth,
                height: polygonHeight,
                transform: transform,
                zIndex: z,
              }">
              <!-- {{ bgImage }} -->
              <div v-show="lighting.length" class="lighting" :style="{ backgroundImage: lighting }" />
            </div>
          </div>
          <div class="bounding-box" :style="{
            left: boundingLeft + 'px',
            top: yMargin + 'px',
            width: boundingRight - boundingLeft + 'px',
            height: pageHeight + 'px',
            cursor: cursor,
          }" @touchstart="onTouchStart" @pointerdown="onPointerDown" @mousedown="onMouseDown" />
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import Matrix from './matrix.js';
// @ts-ignore
import spinner from './spinner.svg';

/**
 * Easing functions for animation curves.
 */
const easeIn = (x: number): number => Math.pow(x, 2)
const easeOut = (x: number): number => 1 - easeIn(1 - x)
const easeInOut = (x: number): number => {
  if (x < 0.5) {
    return easeIn(x * 2) / 2
  }
  return 0.5 + easeOut((x - 0.5) * 2) / 2
};

export default {
  name: 'FlipBook',
  props: {
    /**
     * The array of URLs of the images to display.
     */
    pages: {
      type: Array as () => string[],
      required: true,
    },
    /**
     * The array of URLs of the high-resolution images to display.
     * If not provided, the `pages` array is used.
     */
    pagesHiRes: {
      type: Array as () => string[],
      default: () => [],
    },
    /**
     * The duration of the flip animation in milliseconds.
     */
    flipDuration: {
      type: Number,
      default: 1000,
    },
    /**
     * The duration of the zoom animation in milliseconds.
     */
    zoomDuration: {
      type: Number,
      default: 500,
    },
    /**
     * The array of zoom levels.
     */
    zooms: {
      type: Array as () => number[],
      default: () => [1, 2, 4],
    },
    /**
     * The perspective of the flipbook in pixels.
     */
    perspective: {
      type: Number,
      default: 2400,
    },
    /**
     * The number of polygons to use for the flip animation.
     */
    nPolygons: {
      type: Number,
      default: 10,
    },
    /**
     * The ambient lighting of the flipbook.
     */
    ambient: {
      type: Number,
      default: 0.4,
    },
    /**
     * The gloss of the flipbook.
     */
    gloss: {
      type: Number,
      default: 0.6,
    },
    /**
     * The minimum distance of the swipe gesture in pixels.
     */
    swipeMin: {
      type: Number,
      default: 3,
    },
    /**
     * Whether to display only one page at a time.
     */
    singlePage: {
      type: Boolean,
      default: false,
    },
    /**
     * The direction of the flip animation.
     */
    forwardDirection: {
      validator: (val: string): boolean => val === 'right' || val === 'left',
      default: 'right',
    },
    /**
     * Whether to center the flipbook horizontally.
     */
    centering: {
      type: Boolean,
      default: true,
    },
    /**
     * The starting page of the flipbook.
     */
    startPage: {
      type: Number,
      default: null,
    },
    /**
     * The URL of the loading image.
     */
    loadingImage: {
      type: String,
      default: spinner,
    },
    /**
     * Whether to enable click-to-zoom.
     */
    clickToZoom: {
      type: Boolean,
      default: true,
    },
    /**
     * Whether to enable drag-to-flip.
     */
    dragToFlip: {
      type: Boolean,
      default: true,
    },
    /**
     * The behavior of the mouse wheel.
     */
    wheel: {
      type: String,
      default: 'scroll',
    },
    /**
     * The alt text for the images.
     */
    alt: {
      type: String,
      default: ''
    }
  },
  emits: ['flip-left-start',
    'flip-left-end',
    'flip-right-start',
    'flip-right-end',
    'zoom-start',
    'zoom-end',],
  data() {
    return {
      viewport: null as HTMLDivElement | null,
      viewWidth: 0,
      viewHeight: 0,
      imageWidth: null as number | null,
      imageHeight: null as number | null,
      displayedPages: 1,
      nImageLoad: 0,
      nImageLoadTrigger: 0,
      imageLoadCallback: null as (() => void) | null,
      currentPage: 0,
      firstPage: 0,
      secondPage: 1,
      zoomIndex: 0,
      zoom: 1,
      zooming: false,
      touchStartX: null as number | null,
      touchStartY: null as number | null,
      maxMove: 0,
      activeCursor: null as string | null,
      hasTouchEvents: false,
      hasPointerEvents: false,
      isFlipping: false,
      minX: Infinity,
      maxX: -Infinity,
      preloadedImages: {},
      flip: {
        progress: 0,
        direction: null as string | null,
        frontImage: null as string | null,
        backImage: null as string | null,
        auto: false,
        opacity: 1,
      },
      currentCenterOffset: null as number | null,
      animatingCenter: false,
      startScrollLeft: 0,
      startScrollTop: 0,
      scrollLeft: 0,
      scrollTop: 0,
      loadedImages: {},
    };
  },

  computed: {
    IE(): boolean {
      return typeof navigator !== 'undefined' && /Trident/.test(navigator.userAgent);
    },
    canFlipLeft(): boolean {
      return this.forwardDirection === 'left' ? this.canGoForward : this.canGoBack;
    },
    canFlipRight(): boolean {
      return this.forwardDirection === 'right' ? this.canGoForward : this.canGoBack;
    },
    canZoomIn(): boolean {
      return !this.zooming && this.zoomIndex < (this.zooms_ as number[]).length - 1;
    },
    canZoomOut(): boolean {
      return !this.zooming && this.zoomIndex > 0;
    },
    numPages(): number {
      return this.pages[0] == null ? this.pages.length - 1 : this.pages.length;
    },
    page(): number {
      return this.pages[0] != null ? this.currentPage + 1 : Math.max(1, this.currentPage);
    },
    zooms_(): number[] {
      return this.zooms || [1];
    },
    canGoForward(): boolean {
      return !this.flip.direction && this.currentPage < this.pages.length - this.displayedPages;
    },
    canGoBack(): boolean {
      return (
        !this.flip.direction &&
        this.currentPage >= this.displayedPages &&
        !(this.displayedPages === 1 && !this.pageUrl(this.firstPage - 1))
      );
    },
    leftPage(): number {
      return this.forwardDirection === 'right' || this.displayedPages === 1 ? this.firstPage : this.secondPage;
    },
    rightPage(): number {
      return this.forwardDirection === 'left' ? this.firstPage : this.secondPage;
    },
    showLeftPage(): string | null {
      return this.pageUrl(this.leftPage);
    },
    showRightPage(): string | null {
      return this.pageUrl(this.rightPage) && this.displayedPages === 2 ? this.pageUrl(this.rightPage) : null;
    },
    cursor(): string {
      if (this.activeCursor) return this.activeCursor;
      if (this.IE) return 'auto';
      if (this.clickToZoom && this.canZoomIn) return 'zoom-in';
      if (this.clickToZoom && this.canZoomOut) return 'zoom-out';
      if (this.dragToFlip) return 'grab';
      return 'auto';
    },
    pageScale(): number {
      if (!this.imageWidth || !this.imageHeight) return 1;

      const vw = this.viewWidth / this.displayedPages;
      const xScale = vw / this.imageWidth;
      const yScale = this.viewHeight / this.imageHeight;
      const scale = xScale < yScale ? xScale : yScale;

      return scale < 1 ? scale : 1;
    },
    pageWidth(): number {
      if (!this.imageWidth) return 0;
      return Math.round(this.imageWidth * this.pageScale);
    },
    pageHeight(): number {
      if (!this.imageHeight) return 0;
      return Math.round(this.imageHeight * this.pageScale);
    },
    xMargin(): number {
      return (this.viewWidth - this.pageWidth * this.displayedPages) / 2;
    },
    yMargin(): number {
      return (this.viewHeight - this.pageHeight) / 2;
    },
    polygonWidth(): string {
      let w = this.pageWidth / this.nPolygons;
      w = Math.ceil(w + 1 / this.zoom);
      return `${w}px`;
    },
    polygonHeight(): string {
      return `${this.pageHeight}px`
    },
    polygonBgSize(): string {
      return `${this.pageWidth}px ${this.pageHeight}px`;
    },
    polygonArray(): any {
      return this.makePolygonArray('front').concat(this.makePolygonArray('back'))
    },
    boundingLeft(): number {
      if (this.displayedPages === 1) {
        return this.xMargin;
      }
      const x = this.pageUrl(this.leftPage) ? this.xMargin : this.viewWidth / 2;
      return x < this.minX ? x : this.minX;
    },
    boundingRight(): number {
      if (this.displayedPages === 1) {
        return this.viewWidth - this.xMargin;
      }
      const x = this.pageUrl(this.rightPage) ? this.viewWidth - this.xMargin : this.viewWidth / 2;
      return x > this.maxX ? x : this.maxX;
    },
   
    centerOffset(): number {
      const retval = this.centering ? Math.round(this.viewWidth / 2 - (this.boundingLeft + this.boundingRight) / 2) : 0;

      if (this.currentCenterOffset == null && this.imageWidth != null) {
        this.currentCenterOffset = retval;
      }
      return retval;
    },
    centerOffsetSmoothed(): number {
      return Math.round(this.currentCenterOffset || 0);
    },
    dragToScroll(): boolean {
      return !this.hasTouchEvents;
    },
    scrollLeftMin(): number {
      const w = (this.boundingRight - this.boundingLeft) * this.zoom;
      if (w < this.viewWidth) {
        return (this.boundingLeft + this.centerOffsetSmoothed) * this.zoom - (this.viewWidth - w) / 2;
      }
      return (this.boundingLeft + this.centerOffsetSmoothed) * this.zoom;
    },
    scrollLeftMax(): number {
      const w = (this.boundingRight - this.boundingLeft) * this.zoom;
      if (w < this.viewWidth) {
        return (this.boundingLeft + this.centerOffsetSmoothed) * this.zoom - (this.viewWidth - w) / 2;
      }
      return (this.boundingRight + this.centerOffsetSmoothed) * this.zoom - this.viewWidth;
    },
    scrollTopMin(): number {
      const h = this.pageHeight * this.zoom;
      if (h < this.viewHeight) {
        return this.yMargin * this.zoom - (this.viewHeight - h) / 2;
      }
      return this.yMargin * this.zoom;
    },
    scrollTopMax(): number {
      const h = this.pageHeight * this.zoom;
      if (h < this.viewHeight) {
        return this.yMargin * this.zoom - (this.viewHeight - h) / 2;
      }
      return (this.yMargin + this.pageHeight) * this.zoom - this.viewHeight;
    },
    scrollLeftLimited(): number {
      return Math.min(this.scrollLeftMax, Math.max(this.scrollLeftMin, this.scrollLeft));
    },
    scrollTopLimited(): number {
      return Math.min(this.scrollTopMax, Math.max(this.scrollTopMin, this.scrollTop));
    },
  },
  watch: {
    polygonArray() {

      const front = this.makePolygonArray('front');
      const back = this.makePolygonArray('back');
      // @ts-ignore
      this.minX = Math.min(front.minX, back.minX);
      // @ts-ignore
      this.maxX = Math.max(front.maxX, back.maxX);
      // @ts-ignore
      this.flip.opacity = Math.min(front.flipOpacity, back.flipOpacity);
    },
    currentPage() {
      this.firstPage = this.currentPage;
      this.secondPage = this.currentPage + 1;
      this.preloadImages();
    },
    centerOffset() {
      if (this.animatingCenter) {
        return;
      }

      const animate = () => {
        requestAnimationFrame(() => {
          if(this.currentCenterOffset == null) {
            this.currentCenterOffset = 0
          }
          const rate = 0.1;
          const diff = this.centerOffset - this.currentCenterOffset;
          if (Math.abs(diff) < 0.5) {
            this.currentCenterOffset = this.centerOffset;
            this.animatingCenter = false;
          } else {
            this.currentCenterOffset += diff * rate;
            animate();
          }
        });
      };

      this.animatingCenter = true;
      animate();
    },
    scrollLeftLimited(val) {
      if(!this.viewport) return
      if (this.IE) {
        requestAnimationFrame(() => {
          if(!this.viewport) return
          this.viewport.scrollLeft = val;
        });
      } else {
        this.viewport.scrollLeft = val;
      }
    },
    scrollTopLimited(val) {
      if(!this.viewport) return
      if (this.IE) {
        requestAnimationFrame(() => {
          if(!this.viewport) return
          this.viewport.scrollTop = val;
        });
      } else {
        this.viewport.scrollTop = val;
      }
    },
    pages(after, before) {
      this.fixFirstPage();
      if (!before?.length && after?.length) {
        if (this.startPage > 1 && after[0] === null) {
          this.currentPage += 1;
        }
      }
    },
    startPage(p) {
      this.goToPage(p);
    },
  },
  mounted() {
    this.viewport = this.$refs.viewport as HTMLDivElement;
    window.addEventListener('resize', this.onResize, { passive: true });
    this.onResize();
    this.zoom = this.zooms_[0];
    this.goToPage(this.startPage);
  },

  beforeUnmount() {
    window.removeEventListener('resize', this.onResize);
  },
  methods: {
    onResize(): void {
      /**
       * When the window resizes, set the viewWidth and viewHeight
       * variables to the new dimensions of the viewport element.
       * Also, set the displayedPages variable to 1 if the height is
       * greater than the width, and 2 otherwise.
       */
      
      if (!this.viewport) return;

      this.viewWidth = this.viewport.clientWidth;
      this.viewHeight = this.viewport.clientHeight;
      this.displayedPages = (this.viewWidth > this.viewHeight && !this.singlePage) ? 2 : 1;
      if (this.displayedPages === 2) {
        this.currentPage &= ~1;
      }
      this.fixFirstPage();
      this.minX = Infinity;
      this.maxX = -Infinity;
    },
    fixFirstPage(): void {
      /**
       * If the first page is blank, move to the next page.
       */
      if (this.displayedPages === 1 && this.currentPage === 0 && this.pages.length && !this.pageUrl(0)) {
        this.currentPage += 1;
      }
    },
    pageUrl(page: number, hiRes = false): string | null {
      /**
       * Returns the url of the page.
       * If hiRes is true and the zoom is greater than 1 and not animating,
       * returns the high res image.
       * Otherwise, returns the normal res image.
       */
      if (hiRes && this.zoom > 1 && !this.zooming) {
        const url = this.pagesHiRes[page];
        if (url) return url;
      }

      return this.pages[page] || null;
    },
    pageUrlLoading(page: number, hiRes = false): string | undefined {
      /**
       * Returns the url of the page.
       * If hiRes is true and the zoom is greater than 1 and not animating,
       * returns the high res image.
       * Otherwise, returns the normal res image.
       */
      const url = this.pageUrl(page, hiRes) || undefined;
      if (hiRes && this.zoom > 1 && !this.zooming) return url;
      return url && this.loadImage(url);
    },
    flipLeft(): void {
      /**
       * Flip left
       */
      if (!this.canFlipLeft) return;
      this.flipStart('left', true);
    },
    flipRight(): void {
      /**
       * Flip right
       */
      if (!this.canFlipRight) return;
      this.flipStart('right', true);
    },
    makePolygonArray(face: 'front' | 'back'): Array<[string, string | null, string | null, string, string, number]> {
      /**
       * Constructs an array of polygons for the given face ('front' or 'back').
       * @param face - The face of the page ('front' or 'back').
       * @returns An array of tuples, each containing:
       *   - The key for the polygon.
       *   - The background image URL or null.
       *   - The lighting effect or null.
       *   - The background position.
       *   - The transformation matrix as a string.
       *   - The z-index for stacking order.
       */

      if (!this.flip.direction) return [];

      let progress: number = this.flip.progress;
      let direction: string = this.flip.direction;

      if (this.displayedPages === 1 && direction !== this.forwardDirection) {
        progress = 1 - progress;
        direction = this.forwardDirection;
      }

      this.flip.opacity = this.displayedPages === 1 && progress > 0.7 ? 1 - (progress - 0.7) / 0.3 : 1;

      const image: string | null = face === 'front' ? this.flip.frontImage : this.flip.backImage;
      const polygonWidth: number = this.pageWidth / this.nPolygons;

      let pageX: number = this.xMargin;
      let originRight: boolean = false;

      if (this.displayedPages === 1) {
        if (this.forwardDirection === 'right') {
          if (face === 'back') {
            originRight = true;
            pageX = this.xMargin - this.pageWidth;
          }
        } else if (direction === 'left') {
          if (face === 'back') {
            pageX = this.pageWidth - this.xMargin;
          } else {
            originRight = true;
          }
        } else if (face === 'front') {
          pageX = this.pageWidth - this.xMargin;
        } else {
          originRight = true;
        }
      } else if (direction === 'left') {
        if (face === 'back') {
          pageX = this.viewWidth / 2;
        } else {
          originRight = true;
        }
      } else if (face === 'front') {
        pageX = this.viewWidth / 2;
      } else {
        originRight = true;
      }

      const pageMatrix = new Matrix();
      pageMatrix.translate(this.viewWidth / 2);
      pageMatrix.perspective(this.perspective);
      pageMatrix.translate(-this.viewWidth / 2);
      pageMatrix.translate(pageX, this.yMargin);

      let pageRotation: number = 0;
      if (progress > 0.5) {
        pageRotation = -(progress - 0.5) * 2 * 180;
      }
      if (direction === 'left') {
        pageRotation = -pageRotation;
      }
      if (face === 'back') {
        pageRotation += 180;
      }

      if (pageRotation) {
        if (originRight) { pageMatrix.translate(this.pageWidth); }
        pageMatrix.rotateY(pageRotation);
        if (originRight) { pageMatrix.translate(-this.pageWidth); }
      }

      let theta: number = progress < 0.5 ? progress * 2 * Math.PI : (1 - (progress - 0.5) * 2) * Math.PI;
      if (theta === 0) theta = 1e-9;
      const radius: number = this.pageWidth / theta;

      let radian: number = 0;
      const dRadian: number = theta / this.nPolygons;
      let rotate: number = (dRadian / 2 / Math.PI) * 180;
      let dRotate: number = (dRadian / Math.PI) * 180;

      if (originRight) {
        rotate = -(theta / Math.PI) * 180 + dRotate / 2;
      }

      if (face === 'back') {
        rotate = -rotate;
        dRotate = -dRotate;
      }

      let minX: number = Infinity;
      let maxX: number = -Infinity;
      const polygons: Array<[string, string | null, string | null, string, string, number]> = [];

      for (let i = 0; i < this.nPolygons; i += 1) {
        const bgPos: string = `${(i / (this.nPolygons - 1)) * 100}% 0px`;

        const m = pageMatrix.clone();
        const rad: number = originRight ? theta - radian : radian;
        let x: number = Math.sin(rad) * radius;
        if (originRight) x = this.pageWidth - x;
        let z: number = (1 - Math.cos(rad)) * radius;
        if (face === 'back') z = -z;

        m.translate3d(x, 0, z);
        m.rotateY(-rotate);

        const x0: number = m.transformX(0);
        const x1: number = m.transformX(polygonWidth);

        maxX = Math.max(Math.max(x0, x1), maxX);
        minX = Math.min(Math.min(x0, x1), minX);

        const lighting: string | null = this.computeLighting(pageRotation - rotate, dRotate);

        radian += dRadian;
        rotate += dRotate;

        polygons.push([`${face}${i}`, image, lighting, bgPos, m.toString(), Math.abs(Math.round(z))]);
      }

      this.maxX = maxX;
      this.minX = minX;
      return polygons;
    },
    computeLighting(rot: number, dRotate: number): string {

      const gradients: Array<string> = [];
      const lightingPoints = [-0.5, -0.25, 0, 0.25, 0.5];

      if (this.ambient < 1) {
        const blackness = 1 - this.ambient;
        const diffuse = lightingPoints.map((d) => (1 - Math.cos((rot - dRotate * d) / 180 * Math.PI)) * blackness);
        gradients.push(`
          linear-gradient(to right,
            rgba(0, 0, 0, ${diffuse[0]}),
            rgba(0, 0, 0, ${diffuse[1]}) 25%,
            rgba(0, 0, 0, ${diffuse[2]}) 50%,
            rgba(0, 0, 0, ${diffuse[3]}) 75%,
            rgba(0, 0, 0, ${diffuse[4]}))
        `);
      }

      if (this.gloss > 0 && !this.IE) {
        const DEG = 30;
        const POW = 200;
        const specular = lightingPoints.map((d) =>
          Math.max(
            Math.cos((rot + DEG - dRotate * d) / 180 * Math.PI) ** POW,
            Math.cos((rot - DEG - dRotate * d) / 180 * Math.PI) ** POW
          )
        );
        gradients.push(`
          linear-gradient(to right,
            rgba(255, 255, 255, ${specular[0] * this.gloss}),
            rgba(255, 255, 255, ${specular[1] * this.gloss}) 25%,
            rgba(255, 255, 255, ${specular[2] * this.gloss}) 50%,
            rgba(255, 255, 255, ${specular[3] * this.gloss}) 75%,
            rgba(255, 255, 255, ${specular[4] * this.gloss}))
        `);
      }
      return gradients.join(',');
    },

    async flipStart(direction: string, auto: boolean): Promise<void> {
      if (auto && this.isFlipping) {
        return;
      }

      this.isFlipping = true;

      if (direction !== this.forwardDirection) {
        if (this.displayedPages === 1) {
          this.flip.frontImage = this.pageUrl(this.currentPage - 1);
          this.flip.backImage = null;
        } else {
          this.flip.frontImage = this.pageUrl(this.firstPage);
          this.flip.backImage = this.pageUrl(this.currentPage - this.displayedPages + 1);
        }
      } else {
        if (this.displayedPages === 1) {
          this.flip.frontImage = this.pageUrl(this.currentPage);
          this.flip.backImage = null;
        } else {
          this.flip.frontImage = this.pageUrl(this.secondPage);
          this.flip.backImage = this.pageUrl(this.currentPage + this.displayedPages);
        }
      }

      this.flip.direction = direction;
      this.flip.progress = 0;

      if (auto) {
        await this.delay(100);
      }

      requestAnimationFrame(() => {
        requestAnimationFrame(() => {
          if (direction !== this.forwardDirection) {
            if (this.displayedPages === 2) {
              this.firstPage = this.currentPage - this.displayedPages;
            }
          } else {
            if (this.displayedPages === 1) {
              this.firstPage = this.currentPage + this.displayedPages;
            } else {
              this.secondPage = this.currentPage + 1 + this.displayedPages;
            }
          }
          if (auto) this.flipAuto(true);
        });
      });
      this.isFlipping = false;
    },

    delay(time: number): Promise<void> {
      return new Promise(resolve => setTimeout(resolve, time));
    },

    flipAuto(ease: boolean): void {
      const t0 = Date.now();
      const duration = this.flipDuration * (1 - this.flip.progress);
      const startRatio = this.flip.progress;

      if (this.flip.auto) {
        return;
      }
      this.flip.auto = true;
      // @ts-ignore
      this.$emit(`flip-${this.flip.direction}-start`, this.page);

      let lastUpdateTime = 0;
      const minInterval = 500 / 60;

      const animate = () => {
        requestAnimationFrame(() => {
          const now = Date.now();
          const t = now - t0;
          let ratio = startRatio + t / duration;
          if (ratio > 1) ratio = 1;
          const progress = ease ? easeInOut(ratio) : ratio;

          if (now - lastUpdateTime >= minInterval || ratio >= 1) {
            lastUpdateTime = now;
            this.flip.progress = progress;
          }
          if (ratio < 1) {
            animate();
          } else {
            if (this.flip.direction !== this.forwardDirection) {
              this.currentPage -= this.displayedPages;
            } else {
              this.currentPage += this.displayedPages;
            }
            // @ts-ignore
            this.$emit(`flip-${this.flip.direction}-end`, this.page);
            if (this.displayedPages === 1 && this.flip.direction === this.forwardDirection) {
              this.flip.direction = null;
            } else {
              this.onImageLoad(1, () => {
                this.flip.direction = null;
              });
            }
            this.flip.auto = false;
          }
        });
      };
      animate();
    },

    flipRevert(): void {
      const t0 = Date.now();
      const duration = this.flipDuration * this.flip.progress;
      const startRatio = this.flip.progress;
      this.flip.auto = true;

      const animate = (): void => {
        requestAnimationFrame(() => {
          const t = Date.now() - t0;
          let ratio = startRatio - (startRatio * t) / duration;
          if (ratio < 0) ratio = 0;
          this.flip.progress = ratio;
          if (ratio > 0) {
            animate();
          } else {
            this.firstPage = this.currentPage;
            this.secondPage = this.currentPage + 1;
            if (this.displayedPages === 1 && this.flip.direction !== this.forwardDirection) {
              this.flip.direction = null;
            } else {
              this.onImageLoad(1, () => {
                this.flip.direction = null;
              });
            }
            this.flip.auto = false;
          }
        });
      };
      animate();
    },

    onImageLoad(trigger: number, cb: () => void): void {
      this.nImageLoad = 0;
      this.nImageLoadTrigger = trigger;
      this.imageLoadCallback = cb;
    },

    didLoadImage(ev: Event): void {
      if (this.imageWidth == null) {
        this.imageWidth = (ev.target as HTMLImageElement).naturalWidth;
        this.imageHeight = (ev.target as HTMLImageElement).naturalHeight;
        this.preloadImages();
      }

      if (!this.imageLoadCallback) return;
      if (++this.nImageLoad >= this.nImageLoadTrigger) {
        this.imageLoadCallback();
        this.imageLoadCallback = null;
      }
    },

    zoomIn(zoomAt?: MouseEvent | Touch): void {
      if (!this.canZoomIn) return;
      this.zoomIndex += 1;
      this.zoomTo(this.zooms_[this.zoomIndex], zoomAt);
    },

    zoomOut(zoomAt?: MouseEvent | Touch): void {
      if (!this.canZoomOut) return;
      this.zoomIndex -= 1;
      this.zoomTo(this.zooms_[this.zoomIndex], zoomAt);
    },

    zoomTo(zoom: number, zoomAt?: MouseEvent | Touch): void {
      if(!this.viewport) return      

      let fixedX;
      let fixedY;

      if (zoomAt) {
        const rect = this.viewport.getBoundingClientRect();

        fixedX = zoomAt.pageX - rect.left;
        fixedY = zoomAt.pageY - rect.top;
      } else {
        fixedX = this.viewport.clientWidth / 2;
        fixedY = this.viewport.clientHeight / 2;
      }

      const start = this.zoom;
      const end = zoom;
      const startX = this.viewport.scrollLeft;
      const startY = this.viewport.scrollTop;
      const containerFixedX = fixedX + startX;
      const containerFixedY = fixedY + startY;
      const endX = (containerFixedX / start) * end - fixedX;
      const endY = (containerFixedY / start) * end - fixedY;

      const t0 = Date.now();
      this.zooming = true;
      this.$emit('zoom-start', zoom);

      const animate = () => {
        requestAnimationFrame(() => {
          const t = Date.now() - t0;
          let ratio = t / this.zoomDuration;
          if (ratio > 1 || this.IE) ratio = 1;
          ratio = easeInOut(ratio);
          this.zoom = start + (end - start) * ratio;

          this.scrollLeft = startX + (endX - startX) * ratio;
          this.scrollTop = startY + (endY - startY) * ratio;

          if (t < this.zoomDuration) {
            animate();
          } else {
            this.$emit('zoom-end', zoom);
            this.zooming = false;
            this.zoom = zoom;
            this.scrollLeft = endX;
            this.scrollTop = endY;
          }
        });
      };
      animate();
      if (end > 1) {
        this.preloadImages(true);
      }
    },

    zoomAt(zoomAt: MouseEvent | Touch): void {
      this.zoomIndex = (this.zoomIndex + 1) % this.zooms_.length;
      this.zoomTo(this.zooms_[this.zoomIndex], zoomAt);
    },

    swipeStart(touch: Touch | PointerEvent | MouseEvent): void {
      this.touchStartX = touch.pageX;
      this.touchStartY = touch.pageY;
      this.maxMove = 0;
      if (this.zoom <= 1) {
        if (this.dragToFlip) {
          this.activeCursor = 'grab';
        }
      } else {
        if(!this.viewport) return
        this.startScrollLeft = this.viewport.scrollLeft;
        this.startScrollTop = this.viewport.scrollTop;
        this.activeCursor = 'all-scroll';
      }
    },

    swipeMove(touch: Touch | PointerEvent | MouseEvent): boolean {
      if (!this.dragToFlip) return false;
      if (this.touchStartX == null) return false;
      if (this.touchStartY == null) return false;

      const x = touch.pageX - this.touchStartX;
      const y = touch.pageY - this.touchStartY;
      this.maxMove = Math.max(this.maxMove, Math.abs(x));
      this.maxMove = Math.max(this.maxMove, Math.abs(y));
      if (this.zoom > 1) {
        if (this.dragToScroll) this.dragScroll(x, y);
        return false;
      }
      if (Math.abs(y) > Math.abs(x)) return false;
      this.activeCursor = 'grabbing';
      if (x > 0) {
        if (this.flip.direction == null && this.canFlipLeft && x >= this.swipeMin) {
          this.flipStart('left', false);
        }
        if (this.flip.direction === 'left') {
          this.flip.progress = x / this.pageWidth;
          if (this.flip.progress > 1) this.flip.progress = 1;
        }
      } else {
        if (this.flip.direction == null && this.canFlipRight && x <= -this.swipeMin) {
          this.flipStart('right', false);
        }
        if (this.flip.direction === 'right') {
          this.flip.progress = -x / this.pageWidth;
          if (this.flip.progress > 1) this.flip.progress = 1;
        }
      }
      return true;
    },

    swipeEnd(touch: Touch | PointerEvent | MouseEvent): void {
      if (this.touchStartX == null) return;
      if (this.clickToZoom && this.maxMove < this.swipeMin) {
        this.zoomAt(touch);
      }
      if (this.flip.direction != null && !this.flip.auto) {
        if (this.flip.progress > 0.25) {
          this.flipAuto(false);
        } else {
          this.flipRevert();
        }
      }
      this.touchStartX = null;
      this.activeCursor = null;
      this.currentCenterOffset = 0;
      this.minX = Infinity
      this.maxX = -Infinity
    },

    onTouchStart(ev: TouchEvent): void {
      this.hasTouchEvents = true;
      this.swipeStart(ev.changedTouches[0]);
    },

    onTouchMove(ev: TouchEvent): void {
      if (this.swipeMove(ev.changedTouches[0])) {
        if (ev.cancelable) ev.preventDefault();
      }
    },

    onTouchEnd(ev: TouchEvent): void {
      this.swipeEnd(ev.changedTouches[0]);
    },

    /**
     * Handles the pointer down event.
     * @param {PointerEvent} ev - The pointer event.
     * @returns {void}
     */
    onPointerDown(ev: PointerEvent): void {
      this.hasPointerEvents = true;
      if (this.hasTouchEvents) return;
      if (ev.which && ev.which !== 1) return; // Ignore right-click
      this.swipeStart(ev);
      try {
        (ev.target as Element).setPointerCapture(ev.pointerId);
      } catch (e) { }
    },

    onPointerMove(ev: PointerEvent): void {
      if (!this.hasTouchEvents) this.swipeMove(ev);
    },

    onPointerUp(ev: PointerEvent): void {
      if (this.hasTouchEvents) return;
      this.swipeEnd(ev);
      try {
        (ev.target as Element).releasePointerCapture(ev.pointerId);
      } catch (e) { }
    },

    onMouseDown(ev: MouseEvent): void {
      if (this.hasTouchEvents || this.hasPointerEvents) return;
      if (ev.which && ev.which !== 1) return; // Ignore right-click
      this.swipeStart(ev);
    },

    onMouseMove(ev: MouseEvent): void {
      if (!this.hasTouchEvents && !this.hasPointerEvents) this.swipeMove(ev);
    },

    onMouseUp(ev: MouseEvent): void {
      if (!this.hasTouchEvents && !this.hasPointerEvents) this.swipeEnd(ev);
    },

    dragScroll(x: number, y: number): void {
      this.scrollLeft = this.startScrollLeft - x;
      this.scrollTop = this.startScrollTop - y;
    },

    onWheel(ev: WheelEvent): void {
      if(!this.viewport) return
      if (this.wheel === 'scroll' && this.zoom > 1 && this.dragToScroll) {
        this.scrollLeft = this.viewport.scrollLeft + ev.deltaX;
        this.scrollTop = this.viewport.scrollTop + ev.deltaY;
        if (ev.cancelable) ev.preventDefault();
      }

      if (this.wheel === 'zoom') {
        if (ev.deltaY >= 100) {
          this.zoomOut(ev);
          if (ev.cancelable) ev.preventDefault();
        } else if (ev.deltaY <= -100) {
          this.zoomIn(ev);
          if (ev.cancelable) ev.preventDefault();
        }
      }
    },

    preloadImages(hiRes: boolean = false): void {
      for (let i = this.currentPage - 3; i <= this.currentPage + 3; i++) {
        //console.log('preload ' + this.currentPage + ' ' + i)
        this.pageUrlLoading(i); // This preloads image
      }
      if (hiRes) {
        for (let i = this.currentPage; i < this.currentPage + this.displayedPages; i++) {
          const src = this.pagesHiRes[i];
          if (src) {
            const img = new Image();
            img.src = src;
          }
        }
      }
      //console.log(this.loadedImages)
    },

    goToPage(p: number): void {
      if (p == null || p === this.page) return;
      if (this.pages[0] == null) {
        if (this.displayedPages === 2 && p === 1) {
          this.currentPage = 0;
        } else {
          this.currentPage = p;
        }
      } else {
        this.currentPage = p - 1;
      }
      this.minX = Infinity;
      this.maxX = -Infinity;
      this.currentCenterOffset = this.centerOffset;
    },

    loadImage(url: string): string {
      if (this.imageWidth == null) {
        return url;
      }
      if (this.loadedImages[url]) {
        return url;
      }
      const img = new Image();
      img.onload = () => {
        this.loadedImages[url] = true;
      };
      img.src = url;
      return this.loadingImage || url;
    }
  }
};
</script>

<style scoped>
.viewport {
  -webkit-overflow-scrolling: touch;
  width: 100%;
  height: 100%;
}

.viewport.zoom {
  overflow: scroll;
}

.viewport.zoom.drag-to-scroll {
  overflow: hidden;
}

.flipbook-container {
  position: relative;
  width: 100%;
  height: 100%;
  transform-origin: top left;
  user-select: none;
}

.click-to-flip {
  position: absolute;
  width: 50%;
  height: 100%;
  top: 0;
  user-select: none;
}

.click-to-flip.left {
  left: 0;
}

.click-to-flip.right {
  right: 0;
}

.bounding-box {
  position: absolute;
  user-select: none;
}

.page {
  position: absolute;
  backface-visibility: hidden;
}

.polygon {
  position: absolute;
  top: 0;
  left: 0;
  background-repeat: no-repeat;
  backface-visibility: hidden;
  transform-origin: center left;
}

.polygon.blank {
  background-color: #ddd;
}

.polygon .lighting {
  width: 100%;
  height: 100%;
}
</style>
