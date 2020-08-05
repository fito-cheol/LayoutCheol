<template>
  <div class="v-grid" :style="style">
    <GridItem v-for="v in list"
              :key="v.index"
              :index="v.index"
              :sort="v.sort" 

              :rowNumber="v.rowNumber"
              :start="v.start"
              :widthUnit="widthUnit"
              
              :draggable="draggable"
              :drag-delay="dragDelay"
              :row-count="cellCountPerRow"
              :cell-width="cellWidth"
              :cell-height="cellHeight"
              :window-width="windowWidth"
              :row-shift="rowShift"
              @dragstart="onDragStart"
              @dragend="onDragEnd"
              @drag="onDrag"
              @click="click"
              >
      <slot name="cell"
            :item="v.item"
            :index="v.index"
            :sort="v.sort"
            :remove="() => { removeItem(v) }">
      </slot>
    </GridItem>
  </div>
</template>
<script>
import windowSize from '@/mixins/window_size.js'
import GridItem from './Item.vue'

export default {
  name: 'Grid',
  mixins: [windowSize],
  components: {
    GridItem
  },
  props: {
    items: {
      type: Array,
      default: () => []
    },
    gridWidth: {
      type: Number,
      default: -1
    },
    cellWidth: {
      type: Number,
      default: 80,
    },
    cellHeight: {
      type: Number,
      default: 80
    },
    draggable: {
      type: Boolean,
      default: false
    },
    dragDelay: {
      type: Number,
      default: 0
    },
    sortable: {
       type: Boolean,
       default: false
    },
    center: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      list: [],
    }
  },
  watch: {
    items: {
      // 중첩된 data를 watch하는 법
      // https://ui.toast.com/weekly-pick/ko_20190307/
      handler: function (nextItems = []) {
        let accumulateWidth = 0;
        
        this.list = nextItems.map((item, index) => {
          const widthUnit = item.size
          let {rowNumber, start, newAccumulateWidth} = this.calNextPosition(accumulateWidth, widthUnit)
          accumulateWidth = newAccumulateWidth

          return {
            item:item.content,
            index: index,
            sort: index,
            rowNumber,
            start,
            widthUnit
          }
        })
      },
      // Will fire as soon as the component is created
      immediate: true
    }
  },
  computed: {
    gridResponsiveWidth () {
      if (this.gridWidth < 0) {
        return this.windowWidth
      } else {
        return Math.min(this.windowWidth, this.gridWidth)
      }
    },

    height () {
      return Math.ceil(this.list.length / this.cellCountPerRow) *
        this.cellHeight
    },

    style () {
      return {
        height: this.height + 'px'
      }
    },

    cellCountPerRow () {
      return Math.floor(this.gridResponsiveWidth / this.cellWidth)
    },

    rowShift () {
      if (this.center) {
        let contentWidth = this.list.length * this.cellWidth
        let rowShift = contentWidth < this.gridResponsiveWidth
          ? (this.gridResponsiveWidth - contentWidth) / 2
          : (this.gridResponsiveWidth % this.cellWidth) / 2

        return Math.floor(rowShift)
      }

      return 0
    }
  },
  methods: {
    calNextPosition(accumulateWidth=0, widthUnit=1){
      let {cellCountPerRow} = this;
      let rowNumber = Math.floor(accumulateWidth + widthUnit / cellCountPerRow);
      let isOverFlow = Math.floor(accumulateWidth / cellCountPerRow) != Math.floor(accumulateWidth + widthUnit / cellCountPerRow);
      let emptySpace = isOverFlow ? cellCountPerRow - (accumulateWidth % cellCountPerRow) : 0;

      const startIndex = accumulateWidth + emptySpace
      const newAccumulateWidth = accumulateWidth + emptySpace + widthUnit
      return {rowNumber, start: startIndex, newAccumulateWidth, }
    },
    /* Returns merged event object */
    wrapEvent (other = {}) {
      return {
        datetime: Date.now(),
        items: this.getListClone(),
        ...other
      }
    },
    /* Returns sorted clone of "list" array */
    getListClone () {
      return this.list
        .slice(0)
        .sort((a, b) => {
          return a.sort - b.sort
        })
      //  .map(v => {
      //    return { ...v.item }
      //  })
    },

    removeItem ({ index }) {
      let removeItem = this.list.find(v => v.index === index)
      let removeItemSort = removeItem.sort

      this.list = this.list
        .filter(v => {
          return v.index !== index
        })
        .map(v => {
          let sort = v.sort > removeItemSort
            ? (v.sort - 1)
            : v.sort

          return { ...v, sort }
        })

      this.$emit('remove', this.wrapEvent({ index }))
    },

    onDragStart (event) {
      this.$emit('dragstart', this.wrapEvent(event))
    },

    onDragEnd (event) {
      this.$emit('dragend', this.wrapEvent(event))
    },

    click (event) {
      this.$emit('click', this.wrapEvent(event))
    },

    onDrag (event) {
      if (this.sortable) {
        this.sortList(event.index, event.gridPosition)
      }

      this.$emit('drag', this.wrapEvent({ event }))
    },
    // TODO: 이거 이해하기
    sortList (itemIndex, gridPosition) {
      let targetItem = this.list.find(item => item.index === itemIndex)
      let targetItemSort = targetItem.sort

      /*
        Normalizing new grid position
      */
      gridPosition = Math.max(gridPosition, 0)
      /*
        If you remove this line you can drag items to positions that
        are further than items array length
      */
      gridPosition = Math.min(gridPosition, this.list.length - 1)

      if (targetItemSort !== gridPosition) {
        this.list = this.list.map(item => {
          if (item.index === targetItem.index) {
            return {
              ...item,
              sort: gridPosition
            }
          }

          const { sort } = item

          if (targetItemSort > gridPosition) {
            if (sort <= targetItemSort && sort >= gridPosition) {
              return {
                ...item,
                sort: sort + 1
              }
            }
          }

          if (targetItemSort < gridPosition) {
            if (sort >= targetItemSort && sort <= gridPosition) {
              return {
                ...item,
                sort: sort - 1
              }
            }
          }

          return item
        })

        this.$emit('sort', this.wrapEvent())
      }
    }
  }
}
</script>
<style>
.v-grid {
  display: block;
  position: relative;
  width: 100%;
}
</style>
