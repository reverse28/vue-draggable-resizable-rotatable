<template>
  <div
    class="vdrr"
    :style="style"
    :class="{
      draggable: draggable,
      resizable: resizable,
      rotatable: rotatable,
      active: enabled,
      dragging: dragging,
      resizing: resizing,
      rotating: rotating
    }"
    @mousedown.stop="elmDown"
    @dblclick="fillParent"
  >
    <div
      v-for="(handle, index) in handles"
      v-if="resizable"
      class="handle"
      :key="handle"
      :class="'handle-' + handle"
      :style="{ display: enabled ? 'block' : 'none', cursor: cursorStyle[index]}"
      @mousedown.stop.prevent="handleDown(handle, $event)"
    ></div>
    <slot></slot>
  </div>
</template>

<script>
import { matchesSelectorToParentElements } from '../utils/dom'

export default {
  replace: true,
  name: 'VueDraggableResizableRotatable',
  props: {
    active: {
      type: Boolean, default: false
    },
    deactiveRoot: {
      type: Boolean, default: false
    },
    draggable: {
      type: Boolean, default: true
    },
    resizable: {
      type: Boolean, default: true
    },
    rotatable: {
      type: Boolean, default: true
    },
    w: {
      type: [Number, String],
      default: 200,
      validator: function (val) {
        if (typeof val === 'string') {
          return val === 'auto'
        }
        return val > 0
      }
    },
    h: {
      type: [Number, String],
      default: 200,
      validator: function (val) {
        if (typeof val === 'string') {
          return val === 'auto'
        }
        return val > 0
      }
    },
    minw: {
      type: Number,
      default: 50,
      validator: function (val) {
        return val >= 0
      }
    },
    minh: {
      type: Number,
      default: 50,
      validator: function (val) {
        return val >= 0
      }
    },
    r: {
      type: Number,
      default: 0,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    x: {
      type: Number,
      default: 0,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    y: {
      type: Number,
      default: 0,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    z: {
      type: [ String, Number ],
      default: 'auto',
      validator: function (val) {
        let valid = (typeof val === 'string') ? val === 'auto' : val >= 0
        return valid
      }
    },
    handles: {
      type: Array,
      default: function () {
        return ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml', 'rot']
      },
      validator: function (val) {
        var s = new Set(['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml', 'rot'])

        return new Set(val.filter(h => s.has(h))).size === val.length
      }
    },
    cursors: {
      type: Array,
      default: function () {
        return ['nwse-resize', 'ns-resize', 'nesw-resize', 'ew-resize', 'nwse-resize', 'ns-resize', 'nesw-resize', 'ew-resize']
      }
    },
    dragHandle: {
      type: String,
      default: null
    },
    dragCancel: {
      type: String,
      default: null
    },
    axis: {
      type: String,
      default: 'both',
      validator: function (val) {
        return ['x', 'y', 'both'].indexOf(val) !== -1
      }
    },
    grid: {
      type: Array,
      default: function () {
        return [1, 1]
      }
    },
    parent: {
      type: Boolean, default: false
    },
    maximize: {
      type: Boolean, default: false
    },
    deselectCancel: {
      type: String,
      default: null
    },
     transform_point: {
      type: String,
      default: 'center'
    }

  },

  created () {
    this.parentX = 0
    this.parentW = 9999
    this.parentY = 0
    this.parentH = 9999

    this.lastMouseX = 0
    this.lastMouseY = 0

    this.elmX = 0
    this.elmY = 0

    this.elmW = 0
    this.elmH = 0

    this.lastCenterX = 0
    this.lastCenterY = 0
    this.lastElmX = 0
    this.lastElmY = 0
    this.lastElmW = 0
    this.lastElmH = 0

    this.fixedXName = ''
    this.fixedYName = ''
    this.fixedX = 0
    this.fixedY = 0
  },
  mounted () {
    document.documentElement.addEventListener('mousemove', this.handleMove, true)
    document.documentElement.addEventListener('mousedown', this.deselect, true)
    document.documentElement.addEventListener('mouseup', this.handleUp, true)

    // touch events bindings
    document.documentElement.addEventListener('touchmove', this.handleMove, true)
    document.documentElement.addEventListener('touchend touchcancel', this.deselect, true)
    document.documentElement.addEventListener('touchstart', this.handleUp, true)

    this.elmX = parseInt(this.$el.style.left)
    this.elmY = parseInt(this.$el.style.top)
    this.elmW = this.$el.offsetWidth || this.$el.clientWidth
    this.elmH = this.$el.offsetHeight || this.$el.clientHeight

    // this.reviewDimensions()
  },
  beforeDestroy () {
    document.documentElement.removeEventListener('mousemove', this.handleMove, true)
    document.documentElement.removeEventListener('mousedown', this.deselect, true)
    document.documentElement.removeEventListener('mouseup', this.handleUp, true)

    // touch events bindings removed
    document.documentElement.addEventListener('touchmove', this.handleMove, true)
    document.documentElement.addEventListener('touchend touchcancel', this.deselect, true)
    document.documentElement.addEventListener('touchstart', this.handleUp, true)
  },

  data () {
    return {
      top: this.y,
      left: this.x,
      width: this.w,
      height: this.h,
      rotate: this.r,
      resizing: false,
      dragging: false,
      rotating: false,
      enabled: this.active,
      handle: null,
      zIndex: this.z
    }
  },

  methods: {
    reviewDimensions () {
      if (this.minw > this.w) this.width = this.minw

      if (this.minh > this.h) this.height = this.minh

      if (this.parent) {
        const parentW = parseInt(this.$el.parentNode.clientWidth, 10)
        const parentH = parseInt(this.$el.parentNode.clientHeight, 10)

        this.parentW = parentW
        this.parentH = parentH

        if (this.w > this.parentW) this.width = parentW

        if (this.h > this.parentH) this.height = parentH

        if ((this.x + this.w) > this.parentW) this.width = parentW - this.x

        if ((this.y + this.h) > this.parentH) this.height = parentH - this.y
      }

      this.elmW = this.width
      this.elmH = this.height

      this.$emit('resizing', this.left, this.top, this.width, this.height)
    },
    elmDown (e) {
    
      const target = e.target || e.srcElement

      if (this.$el.contains(target)) {
        if (
          (this.dragHandle && !matchesSelectorToParentElements(target, this.dragHandle, this.$el)) ||
          (this.dragCancel && matchesSelectorToParentElements(target, this.dragCancel, this.$el))) {
          return
        }

        this.reviewDimensions()

        if (!this.enabled) {
        

          if (!this.deactiveRoot) {
           this.enabled = true
           this.$emit('update:active', true)
          }
          
          this.$emit('activated')

        }

        if (this.draggable) {
          this.lastElmX = this.elmX
          this.lastElmY = this.elmY
          this.lastElmW = this.elmW
          this.lastElmH = this.elmH

          this.dragging = true
        }
      }
    },
    deselect (e) {
      let {x: mouseX, y: mouseY} = this.getMouseCoordinate(e)

      this.lastMouseX = mouseX
      this.lastMouseY = mouseY

      const target = e.target || e.srcElement
      const regex = new RegExp('handle-([trmbl]{2})', '')

      if (this.deselectCancel && matchesSelectorToParentElements(target, this.deselectCancel)) {
        return
      }

      if (!this.$el.contains(target) && !regex.test(target.className)) {
        if (this.enabled) {
          this.enabled = false

          this.$emit('deactivated')
          this.$emit('update:active', false)
        }
      }
    },
    handleResizeDown (handle, e) {
      let fixed = {
        x: handle[1] === 'l' ? 'right' : 'left',
        y: handle[0] === 't' ? 'bottom' : 'top'
      }

      let rect = {
        top: this.top,
        right: this.left + this.width,
        bottom: this.top + this.height,
        left: this.left
      }

      let fixedCoordinate = this.rotatedPoint(rect, this.rotate, fixed)

      this.fixedXName = fixed.x
      this.fixedYName = fixed.y

      this.fixedX = fixedCoordinate.x
      this.fixedY = fixedCoordinate.y

      this.lastElmX = this.elmX
      this.lastElmY = this.elmY
      this.lastElmW = this.elmW
      this.lastElmH = this.elmH

      this.resizing = true
    },
    handleRotateDown (handle, e) {
      let {top, left, width, height} = this.$el.getBoundingClientRect()
      this.lastCenterX = window.pageXOffset + left + width / 2
      this.lastCenterY = window.pageYOffset + top + height / 2
      this.rotating = true
    },
    handleDown (handle, e) {
      this.handle = handle

      if (e.stopPropagation) e.stopPropagation()
      if (e.preventDefault) e.preventDefault()

      if (handle === 'rot') {
        this.handleRotateDown(handle, e)
      } else {
        this.handleResizeDown(handle, e)
      }
    },
    fillParent (e) {
      if (!this.parent || !this.resizable || !this.maximize) return

      let done = false

      const animate = () => {
        if (!done) {
          window.requestAnimationFrame(animate)
        }

        if (this.axis === 'x') {
          if (
            this.width === this.parentW && this.left === this.parentX
          ) done = true
        } else if (this.axis === 'y') {
          if (
            this.height === this.parentH && this.top === this.parentY
          ) done = true
        } else if (this.axis === 'both') {
          if (
            this.width === this.parentW &&
            this.height === this.parentH &&
            this.top === this.parentY &&
            this.left === this.parentX
          ) done = true
        }

        if (this.axis === 'x' || this.axis === 'both') {
          if (this.width < this.parentW) {
            this.width++
            this.elmW++
          }

          if (this.left > this.parentX) {
            this.left--
            this.elmX--
          }
        }

        if (this.axis === 'y' || this.axis === 'both') {
          if (this.height < this.parentH) {
            this.height++
            this.elmH++
          }

          if (this.top > this.parentY) {
            this.top--
            this.elmY--
          }
        }

        this.$emit('resizing', this.left, this.top, this.width, this.height)
      }

      window.requestAnimationFrame(animate)
    },
    handleMove (e) {
      if (!this.enabled) {
        return
      }
      let {x: mouseX, y: mouseY} = this.getMouseCoordinate(e)

      if (this.resizing) {
        let rotate = this.rotate
        let width = mouseX - this.lastMouseX
        let height = mouseY - this.lastMouseY
        let c = Math.sqrt(Math.pow(width, 2) + Math.pow(height, 2))
        let angle = this.getAngle(width, height)
        let rad = Math.PI / 180 * (angle - rotate)
        let diffY = Math.round(Math.sin(rad) * c)
        let diffX = Math.round(Math.cos(rad) * c)

        this.elmX = this.lastElmX
        this.elmY = this.lastElmY
        this.elmW = this.lastElmW
        this.elmH = this.lastElmH

        let [handleY, handleX] = this.handle

        if (handleX !== 'm') {
          this.elmW += diffX * (handleX === 'r' ? 1 : -1)
        }

        if (handleY !== 'm') {
          this.elmH += diffY * (handleY === 'b' ? 1 : -1)
        }

        if (this.elmW < this.minw) {
          this.elmW = this.minw
        }

        if (this.elmH < this.minh) {
          this.elmH = this.minh
        }

        this.fixedTo()

        if (this.parent) {
          let change = false

          if (this.elmX < this.parentX) {
            this.elmW -= Math.round(this.parentX - this.elmX)
            change = true
          }

          if (this.elmY < this.parentY) {
            this.elmH -= Math.round(this.parentY - this.elmY)
            change = true
          }

          if (this.elmX + this.elmW > this.parentW) {
            this.elmW = Math.round(this.parentW - this.elmX)
            change = true
          }

          if (this.elmY + this.elmH > this.parentH) {
            this.elmH = Math.round(this.parentH - this.elmY)
            change = true
          }

          if (change) {
            this.fixedTo()
          }
        }

        this.left = (Math.round(this.elmX / this.grid[0]) * this.grid[0])
        this.top = (Math.round(this.elmY / this.grid[1]) * this.grid[1])
        this.width = (Math.round(this.elmW / this.grid[0]) * this.grid[0])
        this.height = (Math.round(this.elmH / this.grid[1]) * this.grid[1])
        this.$emit('resizing', this.left, this.top, this.width, this.height)
      } else if (this.dragging) {
        this.elmX = this.lastElmX
        this.elmY = this.lastElmY
        this.elmW = this.lastElmW
        this.elmH = this.lastElmH
        let diffX = mouseX - this.lastMouseX
        let diffY = mouseY - this.lastMouseY
        let dX = diffX
        let dY = diffY

        if (this.parent) {
          if (this.elmX + dX < this.parentX) {
            diffX = this.parentX - this.elmX
          } else if (this.elmX + this.elmW + dX > this.parentW) {
            diffX = this.parentW - this.elmX - this.elmW
          }
          if (this.elmY + dY < this.parentY) {
            this.mouseOffY = (dY - (diffY = this.parentY - this.elmY))
          } else if (this.elmY + this.elmH + dY > this.parentH) {
            this.mouseOffY = (dY - (diffY = this.parentH - this.elmY - this.elmH))
          }
        }
        this.elmX += diffX
        this.elmY += diffY
        if (this.axis === 'x' || this.axis === 'both') {
          this.left = (Math.round(this.elmX / this.grid[0]) * this.grid[0])
        }
        if (this.axis === 'y' || this.axis === 'both') {
          this.top = (Math.round(this.elmY / this.grid[1]) * this.grid[1])
        }
        this.$emit('dragging', this.left, this.top)
      } else if (this.rotating) {
        let y = mouseY - this.lastCenterY
        let x = mouseX - this.lastCenterX
        this.rotate = (this.getAngle(x, y) + 90) % 360
        this.$emit('rotating', this.rotate)
      }
    },
    handleUp (e) {
      let {x: mouseX, y: mouseY} = this.getMouseCoordinate(e)
      this.lastMouseX = mouseX
      this.lastMouseY = mouseY

      this.handle = null
      if (this.resizing) {
        this.resizing = false
        this.$emit('resizestop', this.left, this.top, this.width, this.height)
      } else if (this.dragging) {
        this.dragging = false
        this.$emit('dragstop', this.left, this.top)
      } else if (this.rotating) {
        this.rotating = false
        this.$emit('rotatestop', this.rotate)
      }

      this.elmX = this.left
      this.elmY = this.top
    },
    getMouseCoordinate (e) {
      if (e.type.indexOf('touch') !== -1) {
        return {
          x: e.changedTouches[0].clientX,
          y: e.changedTouches[0].clientY
        }
      } else {
        return {
          x: e.pageX || e.clientX + document.documentElement.scrollLeft,
          y: e.pageY || e.clientY + document.documentElement.scrollTop
        }
      }
    },
    fixedTo () {
      let rect = {
        top: this.elmY,
        right: this.elmX + this.elmW,
        bottom: this.elmY + this.elmH,
        left: this.elmX
      }

      let fixed = {
        x: this.fixedXName,
        y: this.fixedYName
      }

      let {x: fixedX, y: fixedY} = this.rotatedPoint(rect, this.rotate, fixed)

      let dX = Math.round(this.fixedX - fixedX)
      let dY = Math.round(this.fixedY - fixedY)

      this.elmX += dX
      this.elmY += dY
    },
    rotatedPoint (rect, rotate, point) {
      let {top, right, bottom, left} = rect

      let rad = Math.PI / 180 * rotate
      let cos = Math.cos(rad)
      let sin = Math.sin(rad)

      let originX = (right - left + 1) / 2 + left
      let originY = (bottom - top + 1) / 2 + top

      let x = rect[point.x]
      let y = rect[point.y]

      x -= originX
      y -= originY

      return {
        x: x * cos - y * sin + originX,
        y: x * sin + y * cos + originY
      }
    },
    getAngle (x, y) {
      let theta = Math.atan2(y, x) // range (-PI, PI]
      theta = Math.round(180 / Math.PI * theta) // rads to degs, range (-180, 180]

      if (theta < 0) theta = 360 + theta // range [0, 360)
      return theta
    }
  },

  computed: {
    style () {
      return {
        top: this.top + 'px',
        left: this.left + 'px',
        width: this.width + 'px',
        height: this.height + 'px',
        transform: 'rotate(' + this.rotate + 'deg)',
        'transform-origin': this.transform_point,
        zIndex: this.zIndex
      }
    },
    cursorStyle () {
      const cursorStyleArray = ['nwse-resize', 'ns-resize', 'nesw-resize', 'ew-resize', 'nwse-resize', 'ns-resize', 'nesw-resize', 'ew-resize']

      const ARR_LENGTH = 8
      const STEP = 45

      let startIndex = 0
      let rotate = this.rotate

      if (rotate) {
        startIndex = Math.floor(rotate / STEP)
        if (rotate % STEP > (STEP / 2)) {
          startIndex += 1
        }
      }

      if (startIndex > 1) {
        const len = ARR_LENGTH - startIndex
        return (cursorStyleArray.slice(startIndex, startIndex + len))
          .concat(cursorStyleArray.slice(0, startIndex))
      }

      return cursorStyleArray
    }
  },

  watch: {
    r (val) {
      if (val >= 0) {
        this.rotate = val % 360
      }
    },
    w (val) {
      this.width = val
    },
    h (val) {
      this.height = val
    },
    x (val) {
      this.left = val
    },
    y (val) {
      this.top = val
    },
    active (val) {
      this.enabled = val
    },
    z (val) {
      if (val >= 0 || val === 'auto') {
        this.zIndex = val
      }
    }
  }
}
</script>

<style scoped>
  .vdrr {
    position: absolute;
    box-sizing: border-box;
  }
  .vdrr.dragging {
    user-select: none;
  }
  .vdrr.active {
    content: '';
    position: absolute;
    top: -1px;
    right: -1px;
    bottom: -1px;
    left: -1px;
    border: 1px solid #1baee1;
  }

  .vdrr.active::before {
    content: '';
    width: 100%;
    height: 100%;
    position: absolute;
    border-radius: 50%;
    transform-origin: inherit;
    background: red;
    transform: scale(0.05)
  }

  .handle {
    box-sizing: border-box;
    display: none;
    position: absolute;
    width: 10px;
    height: 10px;
    font-size: 1px;
    border-radius: 5px;
    background: #eee;
    border: 1px solid #1baee1;
  }
  .handle-tl {
    top: -5px;
    left: -5px;
  }
  .handle-tm {
    top: -5px;
    left: 50%;
    margin-left: -5px;
  }
  .handle-tr {
    top: -5px;
    right: -5px;
  }
  .handle-ml {
    top: 50%;
    margin-top: -5px;
    left: -5px;
  }
  .handle-mr {
    top: 50%;
    margin-top: -5px;
    right: -5px;
  }
  .handle-bl {
    bottom: -5px;
    left: -5px;
  }
  .handle-bm {
    bottom: -5px;
    left: 50%;
    margin-left: -5px;
  }
  .handle-br {
    bottom: -5px;
    right: -5px;
  }
  .handle-rot {
    position: absolute;
    top: 0;
    left: 50%;
    margin-top: -20px;
    width: 10px;
    height: 10px;
    transform: translate(-50%, 0);
  }
  .handle-rot:before {
    content: '';
    position: absolute;
    top: 9px;
    left: 50%;
    width: 0;
    height: 6px;
    border-left: 1px solid #1baee1;
    transform: translate(-50%, 0);
  }
  
  @media only screen and (max-width: 768px) {
    /* For mobile phones: */
    [class*="handle-"]:before {
      content: '';
      left: -10px;
      right: -10px;
      bottom: -10px;
      top: -10px;
      position: absolute;
    }
  }
</style>
