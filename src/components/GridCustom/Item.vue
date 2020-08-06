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
    items:{
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
    rowCount: {
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
          console.log(event)
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
      let { zIndex, cellWidth, size, cellHeight, topStatic, leftStatic } = this

      return {
        zIndex,
        width: cellWidth * size + 'px',
        height: cellHeight + 'px',
        transform: `translate3d(${leftStatic}px, ${topStatic}px, 0)`
      }
    },
    newStyle(){
      let { zIndex, cellWidth, cellHeight, topStatic, leftStatic } = this
      
      return {
        zIndex, // 기본 1
        width: cellWidth + 'px', 
        height: cellHeight + 'px',
        transform: `translate3d(${leftStatic}px, ${topStatic}px, 0)` // 기본 좌표 
      }
    },
    
    leftStatic(){
      let accumulSpace = 0
      // let current_row = 0
      let start_x = 0
      let {rowCount, items, sort} = this
      console.log('rowCount', rowCount, items, sort)
      for (let i =0; i <= sort; i++){ //sort는 1부터 시작하니까 -1 해준다
        
        let size = items[i].size
        let rowNumber = Math.floor((accumulSpace + size - 1) / rowCount)
        let isOverFlow = Math.floor(accumulSpace / rowCount) != rowNumber;
        let emptySpace = isOverFlow ? rowCount - (accumulSpace % rowCount): 0;
        let addedSpace = emptySpace + size

        // current_row = rowNumber
        
        start_x = (accumulSpace + emptySpace) % rowCount 
        accumulSpace += addedSpace
        if (sort==1){
          // console.log(sort, size, rowNumber, isOverFlow, emptySpace, addedSpace)
        }
        
      }
      // items에서 내 앞에 있는 cell 정보만 잘라내기
      // 얼마나 앞 셀들이 공간을 차지 했는지 알아내기 rowCount 0이 아니라는 전제
      console.log('accumulSpace', sort, start_x, accumulSpace)
      return this.rowShift + start_x * this.cellWidth
    },
    topStatic(){
      let accumulSpace = 0
      let current_row = 0
      // let start_x = 0
      let {rowCount, items, sort} = this
      for (let i =0; i <= sort; i++){ //sort는 1부터 시작하니까 -1 해준다
        
        let size = items[i].size
        let rowNumber = Math.floor((accumulSpace + size - 1) / rowCount)
        let isOverFlow = Math.floor(accumulSpace / rowCount) != rowNumber;
        let emptySpace = isOverFlow ? rowCount - (accumulSpace % rowCount): 0;
        let addedSpace = emptySpace + size

        current_row = rowNumber
        // start_x = (accumulSpace + emptySpace) % rowCount 
        accumulSpace += addedSpace
      }
      return current_row * this.cellHeight
    },
    left () {
      
      return this.dragging
        ? this.shiftX
        : this.rowShift + (this.sort % this.rowCount) * this.cellWidth
    },

    top () {
      return this.dragging
        ? this.shiftY
        : Math.floor(this.sort / this.rowCount) * this.cellHeight
    }
  },
  methods: {
    wrapEvent (event) {
      let { index, sort } = this
      return { event, index, sort }
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

      gridX = Math.min(gridX, this.rowCount - 1)
      gridY = Math.max(gridY, 0)

      let gridPosition = gridX + gridY * this.rowCount

      const $event = {
        event,
        distanceX,
        distanceY,
        positionX: this.shiftX,
        positionY: this.shiftY,
        index: this.index,
        gridX,
        gridY,
        gridPosition
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
