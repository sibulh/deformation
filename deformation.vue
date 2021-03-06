<template>
  <div class="vdr" id="vdr" :class="{ draggable: draggable, resizable: resizable, active: active , dragging: dragging, resizing: resizing}" @mousedown.stop="elmDown" tabindex="0" @keydown="keydown($event)" :style="style">
    <!-- 如果可改变大小为真 -->
    <template v-if="resizable">
      <div
        class="handle"
        v-for="handle in handles"
        :class="'handle-' + handle"
        @mousedown.stop.prevent="handleDown(handle, $event)"
        :key="handle"
      ></div>
    </template>
    <slot></slot>
  </div>
</template>

<script>
export default {
  replace: true,
  name: 'deformation',
  props: {
    draggable: { // 是否可被拖动
      type: Boolean, default: true
    },
    resizable: { // 是否可改变大小
      type: Boolean, default: true
    },
    w: { // 宽度
      type: Number,
      default: 200,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    h: { // 高度
      type: Number,
      default: 200,
      validator: function (val) {
        return typeof val === 'number'
      }
    },
    minw: { // 最小宽度
      type: Number,
      default: 50,
      validator: function (val) {
        return val > 0
      }
    },
    minh: { // 最小高度
      type: Number,
      default: 50,
      validator: function (val) {
        return val > 0
      }
    },
    x: { // 距父元素左上角X轴偏移量
      type: Number,
      default: 0
    },
    y: { // 距父元素左上角Y轴偏移量
      type: Number,
      default: 0
    },
    zoom: {
      type: Number,
      default: 1
    },
    handles: {
      type: Array,
      default: function () {
        return ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml']
      }
    },
    axis: { // 定义组件可以拖动的轴线
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
    }
  },
  created: function () {
    this.parentX = 0
    this.parentW = 9999
    this.parentY = 0
    this.parentH = 9999
    this.mouseX = 0
    this.mouseY = 0
    this.lastMouseX = 0
    this.lastMouseY = 0
    this.mouseOffX = 0
    this.mouseOffY = 0
    this.elmX = 0
    this.elmY = 0
    this.elmW = 0
    this.elmH = 0
  },
  mounted () {
    // 初始化控件宽高
    if (this.minw > this.w && this.w !== 0) this.width = this.minw
    if (this.minh > this.h && this.h !== 0) this.height = this.minh
    // 判断只能在父窗口内移动的设置
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
    this.$emit('resizing', this.left, this.top, this.width, this.height)
  },
  data: function () {
    return {
      top: this.y,
      left: this.x,
      width: this.w,
      height: this.h,
      resizing: false,
      dragging: false,
      active: false,
      handle: null
    }
  },
  methods: {
    elmDown (e) { // 组件被按下事件
      const target = e.target || e.srcElement
      // 确保事件发生在组件内部
      if (this.$el.contains(target)) {
        if (!this.active) {
          this.lastMouseX = e.pageX || e.clientX + document.documentElement.scrollLeft
          this.lastMouseY = e.pageY || e.clientY + document.documentElement.scrollTop
          document.documentElement.addEventListener('mousemove', this.handleMove, true)
          document.documentElement.addEventListener('mousedown', this.deselect, true)
          document.documentElement.addEventListener('mouseup', this.handleUp, true)
          this.active = true
          this.$emit('activated')
        }
        this.elmX = parseInt(this.$el.style.left)
        this.elmY = parseInt(this.$el.style.top)
        this.elmW = this.$el.offsetWidth || this.$el.clientWidth
        this.elmH = this.$el.offsetHeight || this.$el.clientHeight
        if (this.draggable) {
          this.dragging = true
        }
      }
    },
    deselect (e) { // 取消选择事件
      const target = e.target || e.srcElement
      const regex = new RegExp('handle-([trmbl]{2})', '')
      if (!this.$el.contains(target) && !regex.test(target.className)) {
        if (this.active) {
          document.documentElement.removeEventListener('mousemove', this.handleMove, true)
          document.documentElement.removeEventListener('mousedown', this.deselect, true)
          document.documentElement.removeEventListener('mouseup', this.handleUp, true)
          this.active = false
          this.$emit('deactivated')
        }
      }
    },
    handleDown (handle, e) { // 拖动点按下事件
      // 将handle设置为当前元素
      this.handle = handle
      // 终止事件在传播过程的捕获、目标处理或起泡阶段进一步传播
      if (e.stopPropagation) e.stopPropagation()
      // 取消事件的默认动作。
      if (e.preventDefault) e.preventDefault()
      this.resizing = true
    },
    handleMove (e) {
      this.mouseX = e.pageX || e.clientX + document.documentElement.scrollLeft
      this.mouseY = e.pageY || e.clientY + document.documentElement.scrollTop
      // diffX =  当前鼠标位置 - 上次鼠标位置 + ？？
      let diffX = (this.mouseX - this.lastMouseX + this.mouseOffX) / this.zoom
      let diffY = (this.mouseY - this.lastMouseY + this.mouseOffY) / this.zoom
      this.mouseOffX = this.mouseOffY = 0
      this.lastMouseX = this.mouseX
      this.lastMouseY = this.mouseY
      let dX = diffX
      let dY = diffY
      if (this.resizing) {
        if (this.handle.indexOf('t') >= 0) {
          if (this.elmH - dY < this.minh) this.mouseOffY = (dY - (diffY = this.elmH - this.minh))
          else if (this.elmY + dY < this.parentY) this.mouseOffY = (dY - (diffY = this.parentY - this.elmY))
          this.elmY += diffY
          this.elmH -= diffY
        }
        if (this.handle.indexOf('b') >= 0) {
          if (this.elmH + dY < this.minh) this.mouseOffY = (dY - (diffY = this.minh - this.elmH))
          else if (this.elmY + this.elmH + dY > this.parentH) this.mouseOffY = (dY - (diffY = this.parentH - this.elmY - this.elmH))
          this.elmH += diffY
        }
        if (this.handle.indexOf('l') >= 0) {
          if (this.elmW - dX < this.minw) this.mouseOffX = (dX - (diffX = this.elmW - this.minw))
          else if (this.elmX + dX < this.parentX) this.mouseOffX = (dX - (diffX = this.parentX - this.elmX))
          this.elmX += diffX
          this.elmW -= diffX
        }
        if (this.handle.indexOf('r') >= 0) {
          if (this.elmW + dX < this.minw) this.mouseOffX = (dX - (diffX = this.minw - this.elmW))
          else if (this.elmX + this.elmW + dX > this.parentW) this.mouseOffX = (dX - (diffX = this.parentW - this.elmX - this.elmW))
          this.elmW += diffX
        }
        this.left = (Math.round(this.elmX / this.grid[0]) * this.grid[0])
        this.top = (Math.round(this.elmY / this.grid[1]) * this.grid[1])
        this.width = (Math.round(this.elmW / this.grid[0]) * this.grid[0])
        this.height = (Math.round(this.elmH / this.grid[1]) * this.grid[1])
        this.$emit('resizing', this.left, this.top, this.width, this.height)
      } else if (this.dragging) {
        if (this.parent) {
          if (this.elmX + dX < this.parentX) this.mouseOffX = (dX - (diffX = this.parentX - this.elmX))
          else if (this.elmX + this.elmW + dX > this.parentW) this.mouseOffX = (dX - (diffX = this.parentW - this.elmX - this.elmW))
          if (this.elmY + dY < this.parentY) this.mouseOffY = (dY - (diffY = this.parentY - this.elmY))
          else if (this.elmY + this.elmH + dY > this.parentH) this.mouseOffY = (dY - (diffY = this.parentH - this.elmY - this.elmH))
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
      }
    },
    handleUp (e) {
      this.handle = null
      if (this.resizing) {
        this.resizing = false
        this.$emit('resizestop', this.left, this.top, this.width, this.height)
      }
      if (this.dragging) {
        this.dragging = false
        this.$emit('dragstop', this.left, this.top)
      }
      this.elmX = this.left
      this.elmY = this.top
    },
    keydown (event) {
      switch (event.key) {
        case 'ArrowUp': this.top--; break
        case 'ArrowLeft': this.left--; break
        case 'ArrowDown': this.top++; break
        case 'ArrowRight': this.left++; break
      }
      this.$emit('dragging', this.left, this.top)
    }
  },
  watch: {
    x (newVal) {
      this.left = newVal
    },
    y (newVal) {
      this.top = newVal
    },
    w (newVal) {
      this.width = newVal
    },
    h (newVal) {
      this.height = newVal
    }
  },
  computed: {
    style () {
      const w = this.width === 0? 'auto': this.width + 'px'
      const h = this.height === 0? 'auto': this.height + 'px'
      return {
        top: this.top + 'px',
        left: this.left + 'px',
        width: w,
        height: h
      }
    }
  }
}
</script>

<style scoped>
  .vdr {
    position: absolute;
    box-sizing: border-box;
    user-select: none;
  }
  .draggable:hover {
    cursor: move;
  }
  .handle {
    box-sizing: border-box;
    display: none;
    position: absolute;
    width: 10px;
    height: 10px;
    font-size: 1px;
    background: #EEE;
    border: 1px solid #333;
    z-index: 999;
  }
  .handle-tl {
    top: -10px;
    left: -10px;
    cursor: nw-resize;
  }
  .handle-tm {
    top: -10px;
    left: 50%;
    margin-left: -5px;
    cursor: n-resize;
  }
  .handle-tr {
    top: -10px;
    right: -10px;
    cursor: ne-resize;
  }
  .handle-ml {
    top: 50%;
    margin-top: -5px;
    left: -10px;
    cursor: w-resize;
  }
  .handle-mr {
    top: 50%;
    margin-top: -5px;
    right: -10px;
    cursor: e-resize;
  }
  .handle-bl {
    bottom: -10px;
    left: -10px;
    cursor: sw-resize;
  }
  .handle-bm {
    bottom: -10px;
    left: 50%;
    margin-left: -5px;
    cursor: s-resize;
  }
  .handle-br {
    bottom: -10px;
    right: -10px;
    cursor: se-resize;
  }
  .active .handle {
    display: block;
  }
</style>