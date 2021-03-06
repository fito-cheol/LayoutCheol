<template>
  <div ref="self"
       :class="className"
       :style="style"
       @mousedown="mousedown"
       @touchstart.stop="mousedown">
    <slot/>
  </div>
</template>

<script>
const CLICK_PIXEL_DISTANCE = 4

export default {
  name: 'GridItem',
  props: {
    item:{
      type: String
    },
    list:{
      type: Array
    },
    size:{
      type: Number
    },
    index: {
      type: Number
    },
    sort: {
      type: Number
    },
    cellWidth: {
      type: Number
    },
    cellHeight: {
      type: Number
    },
    cellCountPerRow: {
      type: Number
    },
    rowShift: {
      type: Number,
      default: 0
    },
    draggable: {
      type: Boolean
    },
    dragDelay: {
      type: Number,
      default: 0
    }
  },
  data () {
    return {
      animate: true,
      dragging: false,

      shiftStartX: 0,
      shiftStartY: 0,

      mouseMoveStartX: 0,
      mouseMoveStartY: 0,

      shiftX: 0,
      shiftY: 0,

      timer: null,

      zIndex: 1
    }
  },
  mounted () {
    this.$refs.self
      .addEventListener('transitionend', (event) => {
        if (!this.dragging) {
          event // event error 안내려고 장식용
          this.zIndex = 1
        }
      }, false)
  },
  computed: {
    className () {
      let { animate, dragging } = this

      return [
        'v-grid-item-wrapper',
        {
          'v-grid-item-animate': animate, // 드래그 하는 동안 비활성화 되는 class 
          'v-grid-item-dragging': dragging // 드래그 하는 동안 활성화 되는 class
        }
      ]
    },
    style () {
      let { zIndex, cellWidth, size, cellHeight, top, left } = this

      return {
        zIndex,
        width: cellWidth * size + 'px',
        height: cellHeight + 'px',
        transform: `translate3d(${left}px, ${top}px, 0)`
      }
    },
    left(){
      let {startPosition} = this.calPosition()
      return this.dragging
        ? this.shiftX
        : this.rowShift + startPosition * this.cellWidth
    },
    top(){
      let {currentRow} = this.calPosition()
      return this.dragging
        ? this.shiftY
        :currentRow * this.cellHeight
    },
   
  },
  methods: {
    wrapEvent (event) {
      let { index, sort } = this
      return { event, index, sort }
    },
    calPosition(){
      let position = 0
      let currentRow = 0
      let startPosition = 0
      let {cellCountPerRow, list, sort} = this
      for (let i =0; i <= sort && i < list.length; i++){ //sort는 1부터 시작하니까 -1 해준다
        let size = list[i].size
        let startCellRowNumber = Math.floor(position / cellCountPerRow)
        let endCellRowNumber = Math.floor((position + size - 1) / cellCountPerRow)
        let isCellExceedRow = startCellRowNumber != endCellRowNumber;
        let emptySpace = isCellExceedRow ? cellCountPerRow - (position % cellCountPerRow): 0;
        let addedCellSize = emptySpace + size

        currentRow = endCellRowNumber
        startPosition = (position + emptySpace) % cellCountPerRow 
        position += addedCellSize
      }
      return {currentRow, startPosition}
    },
    dragStart (event) {
      let e = event.touches ? event.touches[0] : event

      this.zIndex = 2

      this.shiftX = this.shiftStartX = this.left
      this.shiftY = this.shiftStartY = this.top

      this.mouseMoveStartX = e.pageX
      this.mouseMoveStartY = e.pageY

      this.animate = false
      this.dragging = true

      document.addEventListener('mousemove', this.documentMouseMove)
      document.addEventListener('touchmove', this.documentMouseMove)

      this.$emit('dragstart', this.wrapEvent(event))
    },

    drag (event) {
      let e = event.touches ? event.touches[0] : event

      let distanceX = e.pageX - this.mouseMoveStartX
      let distanceY = e.pageY - this.mouseMoveStartY

      this.shiftX = distanceX + this.shiftStartX
      this.shiftY = distanceY + this.shiftStartY

      let gridX = Math.round(this.shiftX / this.cellWidth)
      let gridY = Math.round(this.shiftY / this.cellHeight)

      gridX = Math.min(gridX, this.cellCountPerRow - 1)
      gridY = Math.max(gridY, 0)
      
      let position = gridX + gridY * this.cellCountPerRow

      const $event = {
        event,
        distanceX,
        distanceY,
        positionX: this.shiftX,
        positionY: this.shiftY,
        index: this.index,
        gridX,
        gridY,
        position
      }

      this.$emit('drag', $event)
    },

    mousedown (event) {
      if (this.draggable) {
        this.timer = setTimeout(() => {
          this.dragStart(event)
        }, this.dragDelay)

        document.addEventListener('mouseup', this.documentMouseUp)
        document.addEventListener('touchend', this.documentMouseUp)
      }
    },

    documentMouseMove (event) {
      if (this.draggable && this.dragging) {
        this.drag(event)
      }
    },

    documentMouseUp (event) {
      if (this.timer) {
        clearTimeout(this.timer)
        this.timer = null
      }

      let dx = this.shiftStartX - this.shiftX
      let dy = this.shiftStartY - this.shiftY

      let distance = Math.sqrt(dx * dx + dy * dy)

      this.animate = true
      this.dragging = false
      this.mouseMoveStartX = 0
      this.mouseMoveStartY = 0
      this.shiftStartX = 0
      this.shiftStartY = 0

      document.removeEventListener('mousemove', this.documentMouseMove)
      document.removeEventListener('touchmove', this.documentMouseMove)

      document.removeEventListener('mouseup', this.documentMouseUp)
      document.removeEventListener('touchend', this.documentMouseUp)

      let $event = this.wrapEvent(event)

      if (distance < CLICK_PIXEL_DISTANCE) {
        this.$emit('click', $event)
      }

      this.$emit('dragend', $event)
    }
  }
}
</script>

<style>
.v-grid-item-wrapper {
  display: block;
  position: absolute;
  box-sizing: border-box;

  left: 0;
  top: 0;

  user-select: none;
  transform: translate3d(0px, 0px, 0px);

  z-index: 1;
}

.v-grid-item-animate {
  transition: transform 800ms ease;
}

</style>
