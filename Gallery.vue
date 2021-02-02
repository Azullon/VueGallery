<template>
  <div class="gallery">
    <div
      class="gallery__prompt prompt"
      @mouseenter="isPromptHovered = true"
      @mouseleave="isPromptHovered = false"
    >
      <transition name="drop-down">
        <div
          class="prompt__content"
          v-if="isGalleryHovered || isPromptHovered || isTouchDevice"
        >
          {{ $t('clickToOpenFullscreen') }}
        </div>
      </transition>
    </div>

    <canvas
      :width="blockWidth"
      :height="blockHeight"
      @mousewheel.stop.prevent="onScroll"
      @touchstart="onTouchStart"
      @touchmove="onTouchMove"
      @touchend="onTouchEnd"
      @mouseenter="onHover"
      @mouseleave="onUnhover"
      @click="onClick"
    >
    </canvas>

    <fullscreen-image
      v-if="fullscreenImageIndex != -1"
      :isVisible="isFullscreenOpened"
      :image="images[fullscreenImageIndex].before"
      @closed="onFullscreenClosed"
    ></fullscreen-image>
  </div>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component'
import { Prop } from 'vue-property-decorator'
import assets from '@/assets'
import FullscreenImage from './FullscreenImage.vue'
import { isTouchDevice } from '@/utils'

interface Delta {
  deltaX: number
  deltaY: number
}

@Options({
  components: {
    FullscreenImage,
  },
  name: 'Gallery',
})
export default class Gallery extends Vue {
  //Props
  @Prop() images: { before: string; after: string }[] = []

  //Assets
  assets = assets

  //Hooks
  beforeMount() {
    console.log('Gallery')
    this.loadedImages = new Array(this.images.length)
    for (let i = 0; i < this.images.length; i++) {
      let image = this.images[i]
      let imageObj = new Image()
      imageObj.src = image.before
      imageObj.onload = () => {
        if (this.loadedImages) this.loadedImages[i] = imageObj
      }
    }
  }
  mounted() {
    this.trackSize()
    this.startTicker()
  }
  unmounted() {
    this.untrackSize()
  }

  get galleryDOMElement() {
    return this.$el
  }
  get canvasDOMElement() {
    return this.$el.children[1]
  }
  canvasContext?: CanvasRenderingContext2D

  //Resizing logic
  blockHeight = 1
  blockWidth = 1
  onWindowResize: any
  trackSize() {
    let trackFn = () => {
      this.blockHeight = this.galleryDOMElement.clientHeight ?? 1
      this.blockWidth = this.galleryDOMElement.clientWidth ?? 1
    }
    this.onWindowResize = trackFn
    window.addEventListener('resize', trackFn)
    trackFn()
  }
  untrackSize() {
    window.removeEventListener('resize', this.onWindowResize)
  }

  //Automatic rotation
  offset = 0
  rotationSpeed = 2
  isTickerRunning = false
  lastTick?: Date
  startTicker() {
    this.canvasContext = this.canvasDOMElement.getContext('2d')
    this.isTickerRunning = true
    this.lastTick = new Date()
    requestAnimationFrame(this.tick)
  }
  stopTicker() {
    this.isTickerRunning = false
  }
  tick() {
    if (this.isTickerRunning) {
      this.offset += this.rotationSpeed
      this.renderCanvas()
      requestAnimationFrame(this.tick)
    }
  }

  //Image rendering logic
  get imagesCount() {
    return Math.ceil(
      1 + (this.blockWidth - this.firstImageWidth) / this.blockHeight
    )
  }
  get firstImageWidth() {
    return this.blockHeight - (this.offset % this.blockHeight)
  }
  get lastImageWidth() {
    return (this.blockWidth - this.firstImageWidth) % this.blockHeight
  }
  get imageIndexes() {
    let firstIndex =
      Math.floor(this.offset / this.blockHeight) % this.images.length

    let result = new Array(this.imagesCount)
    result[0] = firstIndex
    for (let i = 1; i < result.length; i++) {
      result[i] = (result[i - 1] + 1) % this.images.length
    }
    return result
  }

  loadedImages?: HTMLImageElement[]
  areAllImageLoaded() {
    if (!this.loadedImages) return false
    for (const image of this.loadedImages) {
      if (!image) return false
    }
    return true
  }
  calculateCoverSize(
    imageObj: HTMLImageElement,
    partX: number,
    partWidth: number
  ) {
    let result = {
      x: 0,
      y: 0,
      width: 1,
      height: 1,
    }
    let insertedBlockSize = {
      height: this.blockHeight,
      width: this.blockWidth,
    }
    insertedBlockSize.width *= imageObj.naturalHeight / insertedBlockSize.height
    insertedBlockSize.height = imageObj.naturalHeight
    if (insertedBlockSize.width > imageObj.naturalWidth) {
      insertedBlockSize.height *=
        imageObj.naturalWidth / insertedBlockSize.width
      insertedBlockSize.width = imageObj.naturalWidth
      result.y = (imageObj.naturalHeight - insertedBlockSize.height) / 2
    } else {
      result.x = (imageObj.naturalWidth - insertedBlockSize.width) / 2
    }
    result.x += (partX / this.blockWidth) * insertedBlockSize.width
    result.height = insertedBlockSize.height
    result.width = insertedBlockSize.width * (partWidth / this.blockWidth)
    return result
  }
  renderCanvas() {
    if (!this.loadedImages) return
    if (!this.areAllImageLoaded()) return
    if (!this.canvasContext) return
    let currentPosition = 0
    let imageIndexes = this.imageIndexes

    for (let i = 0; i < imageIndexes.length; i++) {
      let imageObj = this.loadedImages[imageIndexes[i]]
      let imageWidth = this.blockHeight
      if (i == 0) imageWidth = this.firstImageWidth
      else if (i == imageIndexes.length - 1) imageWidth = this.lastImageWidth
      let cs = this.calculateCoverSize(imageObj, currentPosition, imageWidth)
      this.canvasContext.drawImage(
        imageObj,
        cs.x,
        cs.y,
        cs.width,
        cs.height,
        currentPosition,
        0,
        imageWidth,
        this.blockHeight
      )
      currentPosition += imageWidth
    }
  }

  //Hover logic
  isGalleryHovered = false
  isPromptHovered = false
  isTouchDevice = isTouchDevice()
  onHover() {
    this.isGalleryHovered = true
    //this.stopTicker()
  }
  onUnhover() {
    setTimeout(() => {
      if (this.isPromptHovered) return
      this.isGalleryHovered = false
    }, 0)
  }

  //Fullscreen
  fullscreenImageIndex = -1
  isFullscreenOpened = false
  onFullscreenClosed() {
    this.isFullscreenOpened = false
    this.$nextTick(() => this.onUnhover())
    this.startTicker()
  }
  onClick(event: MouseEvent) {
    if (event.offsetX < this.firstImageWidth) {
      this.fullscreenImageIndex = this.imageIndexes[0]
    } else {
      this.fullscreenImageIndex = this.imageIndexes[
        1 +
          Math.floor((event.offsetX - this.firstImageWidth) / this.blockHeight)
      ]
    }
    this.stopTicker()
    this.isFullscreenOpened = true
  }

  //Manual rotation logic
  onScroll(event: WheelEvent | Delta) {
    this.offset += (event.deltaX + event.deltaY) * this.rotationSpeed
    if (this.offset < 0) this.offset += this.images.length * this.blockHeight
  }
  isTouchStarted = false
  lastTouchX = 0
  totalDelta = 0
  onTouchStart(event: TouchEvent) {
    this.stopTicker()
    this.isTouchStarted = true
    this.lastTouchX = event.touches[0].screenX
  }
  onTouchMove(event: TouchEvent) {
    if (event.changedTouches.length > 0) {
      event.changedTouches[0]
      let delta = this.lastTouchX - event.changedTouches[0].screenX
      this.onScroll({
        deltaY: 0,
        deltaX: delta * 2,
      })
      this.totalDelta += delta
      this.lastTouchX = event.changedTouches[0].screenX
      this.renderCanvas()
    }
  }
  onTouchEnd(event: TouchEvent) {
    this.startTicker()
    this.onTouchMove(event)
  }
}
</script>

<style lang="scss" scoped>
.gallery {
  overflow: hidden;
  &__box {
    display: inline-flex;
    width: calc(100% + 100vh);
  }

  &__image {
    &:hover {
      //z-index: 1001;
    }
  }
}

.prompt {
  margin: 16px auto -60px;
  width: fit-content;
  height: 44px;
  &__content {
    z-index: 1;
    padding: 0 16px;
    line-height: 44px;
    backdrop-filter: blur(20px);
    background: rgba($color: #000000, $alpha: 0.4);
    text-align: center;
    border-radius: 8px;
    font-size: 24px;
    color: white;
  }
}

@media (max-width: 640px) {
  .prompt {
    margin: 16px auto -46px;
    height: 30px;
    &__content {
      font-size: 16px;
      line-height: 30px;
    }
  }
}

.masked-image {
  overflow: hidden;
  &__content {
  }
}

.screen-covering-image {
  width: 100vw;
  height: 100vh;
  background-size: cover;
  background-position: center;
}

.drop-down-enter-active {
  animation: drop-down-enter 0.3s;
}
.drop-down-leave-active {
  animation: drop-down-leave 0.3s;
}

@keyframes drop-down-enter {
  from {
    transform: translateY(-100%);
  }
  to {
    transform: translateY(0);
  }
}
@keyframes drop-down-leave {
  from {
    transform: translateY(0);
  }
  to {
    transform: translateY(-200%);
  }
}
</style>
