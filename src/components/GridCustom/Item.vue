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
    // 단위 ms : 왼클릭을 얼마나 눌려야 dragEvent가 발생하는지 측정
    // 실수를 방지하기 위해서 100ms를 주던지 0을 주던지 별로 필요없는 옵션
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
    // TODO: transitioned라는 event가 던제 발생하는지 알아보기
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
      let { zIndex, cellWidth, cellHeight, top, left } = this

      return {
        zIndex, // 기본 1
        width: cellWidth + 'px', 
        height: cellHeight + 'px',
        transform: `translate3d(${left}px, ${top}px, 0)` // 기본 좌표 
      }
    },
    // sort cellCountPerRow cellWidth cellHeight 
    left () {
      return this.dragging
        ? this.shiftX // 드래그시
        : this.rowShift + (this.sort % this.cellCountPerRow) * this.cellWidth // 기본 세팅
    },

    top () {
      return this.dragging
        ? this.shiftY // 드래그시
        : Math.floor(this.sort / this.cellCountPerRow) * this.cellHeight // 기본 세팅
    }
  },
  methods: {
    wrapEvent (event) {
      let { index, sort } = this
      return { event, index, sort }
    },

    dragStart (event) {
      // drag 시작할 때 기본값 세팅
      let e = event.touches ? event.touches[0] : event

      this.zIndex = 2
      // TODO: this.left와 top은 computed에서 나왔음 나중에 확인해볼 것
      this.shiftX = this.shiftStartX = this.left
      this.shiftY = this.shiftStartY = this.top
      // TODO: e.pageX와 this.left가 차이가 나는지 궁금
      this.mouseMoveStartX = e.pageX
      this.mouseMoveStartY = e.pageY

      this.animate = false
      this.dragging = true

      // 나중에 드래그 끝날 때 없애주는 이벤트 등록
      document.addEventListener('mousemove', this.documentMouseMove)
      document.addEventListener('touchmove', this.documentMouseMove)

      this.$emit('dragstart', this.wrapEvent(event))
    },

    drag (event) {
      let e = event.touches ? event.touches[0] : event

      let distanceX = e.pageX - this.mouseMoveStartX // 드래그에서 얼마나 벗어났나
      let distanceY = e.pageY - this.mouseMoveStartY 

      this.shiftX = distanceX + this.shiftStartX // 화면 좌측 상단에서 얼마나 옮겨야하는지
      this.shiftY = distanceY + this.shiftStartY

      let gridX = Math.round(this.shiftX / this.cellWidth) // 정수형 좌표
      let gridY = Math.round(this.shiftY / this.cellHeight)

      // TODO: cellCountPerRow가 아니라 colCount라고 이름을 바꿔야 될거 같음
      //      근데 cellCountPerRow를 gridY에다가 곱하는게 말이 되는데 뭘까
      gridX = Math.min(gridX, this.cellCountPerRow - 1)
      gridY = Math.max(gridY, 0)

      // gridPositiond은 Layout.vue의 sortList라는 method에서 쓰임
      let gridPosition = gridX + gridY * this.cellCountPerRow

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

// transition 참고 https://thoughtbot.com/blog/transitions-and-transforms#:~:text=Transitions%20are%20the%20grease%20in,making%20it%20smooth%20and%20gradual.
.v-grid-item-animate {
  transition: transform 800ms ease;
}

</style>
