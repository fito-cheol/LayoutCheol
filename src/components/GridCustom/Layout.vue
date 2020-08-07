<template>
  <div class="v-grid" :style="style">
    <GridItem v-for="v in list"
              :key="v.index"
              :index="v.index"
              :size="v.size"
              :sort="v.sort"
              :list="list"
              :draggable="draggable"
              :drag-delay="dragDelay"
              :cellCountPerRow="cellCountPerRow"
              :cell-width="cellWidth"
              :cell-height="cellHeight"
              :window-width="windowWidth"
              :row-shift="rowShift"
              @dragstart="onDragStart"
              @dragend="onDragEnd"
              @drag="onDrag"
              @click="click">
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
    centerAlign: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      list: []
    }
  },
  watch: {
    items: {
      handler: function (nextItems = []) {
        this.list = nextItems.map((item, index) => {
          return {
            item: item.content,
            size: item.size,
            index: index,
            sort: index,
          }
        })
      },
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
      if (this.centerAlign) {
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
    
    sortList (itemIndex, gridPosition) {
      let targetItem = this.list.find(item => item.index === itemIndex)
      let targetItemSort = targetItem.sort
      let startPointList = this.getStartPointList()
      let startPoint = startPointList[targetItemSort]
      
      gridPosition = this.nomalizeGridPosition(gridPosition, startPointList)
      

      
      // sort 의 값만 바꿔준다
      
      let isDragChangePosition =  startPoint !== gridPosition
      if (isDragChangePosition) {
        this.list = this.list.map(item => {

          if (item.index === targetItem.index) { 
            let new_sort = 0
            if (gridPosition== -1){
              return {
                  ...item,
                  sort: new_sort,
                }
            }
            for (let sort_index in startPointList){
              sort_index = Number(sort_index)
              // 마지막 인덱스가 넘어가면 제일 뒤
              if (sort_index+1 == startPointList.length){
                new_sort = sort_index;
                break;
              }
              // 상자 크기 내에 있으면 바꿔줌
              if (startPointList[sort_index] <= gridPosition && gridPosition < startPointList[sort_index+1]){
                new_sort = sort_index;
                break;
              }
            }
            return {
                  ...item,
                  sort: new_sort,
                }
          }

          const { sort, size } = item
          let comparedStartPoint = startPointList[sort]
          
          let isDraggedLeft = gridPosition < startPoint 
          if (isDraggedLeft) { 
            // 사이즈가 2면 comparedStartPoint >= gridPosition 가 아니라 comparedStartPoint+1 >= gridPosition이 되어야한다
            if (comparedStartPoint <= startPoint && comparedStartPoint + size -1 >= gridPosition) {
              
              return {
                ...item,
                sort: sort + 1,
              }
            }
          }
          let isDraggedRight = gridPosition > startPoint  
          if (isDraggedRight) {
            if (comparedStartPoint > startPoint && comparedStartPoint <= gridPosition) {
              return {
                ...item,
                sort: sort - 1,
              }
            }
          }

          return item
        })

        this.$emit('sort', this.wrapEvent())
      }
      // sort 순서대로 다시 정렬을 해줘야한다
      this.list.sort(function(a,b){
        return Number(a.sort) > Number(b.sort) ? 1: -1
      })

    },
    nomalizeGridPosition(gridPosition, startPointList){
      gridPosition = Math.max(gridPosition, -1)
      /*
        If you remove this line you can drag items to positions that
        are further than items array length
      */
      gridPosition = Math.min(gridPosition, startPointList[this.list.length - 1])
      return gridPosition
    },
    getStartPointList(){
      let accumulationList = []
      let centerPointList = []
      let accumulSpace = 0
      let {cellCountPerRow, list} = this
      for (let i =0; i < list.length; i++){ //sort는 1부터 시작하니까 -1 해준다
        
        let size = list[i].size
        let rowNumber = Math.floor((accumulSpace + size - 1) / cellCountPerRow)
        let isOverFlow = Math.floor(accumulSpace / cellCountPerRow) != rowNumber;
        let emptySpace = isOverFlow ? cellCountPerRow - (accumulSpace % cellCountPerRow): 0;
        let addedSpace = emptySpace + size

        accumulationList.push(accumulSpace)
        centerPointList.push(Number(accumulSpace+size/2))
        accumulSpace += addedSpace
      }
      return accumulationList
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
