<template>
  <transition name="fade">
    <div
      v-if="isVisible"
      class="fullscreen-image"
      @mousemove="onMouseMove"
      @touchmove="onTouchMove"
    >
      <img
        class="fullscreen-image__main-content"
        :style="`transform: translateY(${-imageY}px);`"
        :src="image"
        alt=""
      />
      <img
        class="fullscreen-image__close-button"
        src="../assets/minimize.svg"
        alt=""
        @click.stop.prevent="$emit('closed')"
      />
    </div>
  </transition>
</template>

<script lang="ts">
import { Options, Vue } from 'vue-class-component'
import { Prop, Watch } from 'vue-property-decorator'

@Options({
  name: 'FullscreenImage',
})
export default class FullscreenImage extends Vue {
  @Prop() image = ''
  @Prop() isVisible = false

  mounted() {
    console.log('Fullscreen image')
    this.onVisibilityChange(this.isVisible)
  }

  @Watch('isVisible')
  onVisibilityChange(newVal: boolean) {
    if (newVal) {
      this.onWindowResize = () => this.updateImageHeight()
      window.addEventListener('resize', this.onWindowResize)
      document.documentElement.requestFullscreen()
      this.onWindowResize()
    } else {
      try {
        document.exitFullscreen()
      } catch {}
      window.removeEventListener('resize', this.onWindowResize)
    }
  }

  onWindowResize: any

  updateImageHeight() {
    let elem: any = document.querySelector('.fullscreen-image__main-content')
    if (elem) {
      this.imageHeight = elem.clientHeight
    } else requestAnimationFrame(() => this.updateImageHeight())
  }

  imageHeight = 0
  mousePositionY = 0

  onMouseMove(event: MouseEvent) {
    this.mousePositionY = event.y
  }

  onTouchMove(event: TouchEvent) {
    if (event.changedTouches.length > 0) {
      this.mousePositionY = event.changedTouches[0].clientY
    }
  }

  get imageY() {
    if (this.imageHeight <= window.innerHeight)
      return -(window.innerHeight - this.imageHeight) / 2
    else {
      let range = this.imageHeight - window.innerHeight
      let scrollPercent = this.mousePositionY / window.innerHeight
      if (scrollPercent < 0.1) scrollPercent = 0
      else if (scrollPercent > 0.9) scrollPercent = 1
      else scrollPercent = (scrollPercent - 0.1) * 1.25
      return scrollPercent * range
    }
    return 0
  }
}
</script>

<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.fullscreen-image {
  z-index: 900;
  position: fixed;
  height: 100vh;
  width: 100vw;
  top: 0;
  left: 0;
  background: #212529;

  &__main-content {
    width: 100%;
  }

  &__close-button {
    position: absolute;
    top: 40px;
    right: 40px;

    height: 7.5%;
    max-height: 60px;
    width: 7.5%;
    max-width: 60px;

    opacity: 0.7;

    &:hover {
      opacity: 1 !important;
      cursor: pointer;
      transition: 0.2s;
    }

    &:not(:hover) {
      opacity: 0.7;
      transition: 0.2s;
    }
  }
}
</style>
