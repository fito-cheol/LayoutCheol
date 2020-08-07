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

    // 중앙 정렬시 x방향으로 얼만큼 이동하는지 (단위 px일듯) 
    // padding의 개념이라 보면 될듯 왜 style에 넣을 수도 있음
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
        this.sortList(event.index, event.position)
      }
      this.$emit('drag', this.wrapEvent({ event }))
    },
    
    sortList (itemIndex, dragPosition) {
      let targetItem = this.list.find(item => item.index === itemIndex)
      let targetItemSort = targetItem.sort
      let startPositionList = this.getStartPositionList()
      let startPosition = startPositionList[targetItemSort]
      
      dragPosition = this.nomalizePosition(dragPosition, startPositionList)
      
      // sort 의 값만 바꿔준다
      let isDragChangePosition =  startPosition !== dragPosition
      if (isDragChangePosition) {
        this.list = this.list.map(item => {

          const isDraggedItem = item.index === targetItem.index
          if (isDraggedItem) { 
            let new_sort = 0
            // 가장 앞쪽일 때
            const isMostFront = dragPosition == -1
            if (isMostFront){
              return {
                  ...item,
                  sort: new_sort,
                }
            }
            for (let sort_index in startPositionList){
              sort_index = Number(sort_index)
              
              // 가장 뒤쪽 일 때
              const isMostEnd = sort_index+1 == startPositionList.length
              if (isMostEnd){
                new_sort = sort_index;
                break;
              }
              
              const currentStartPosition = startPositionList[sort_index] 
              const nextStartPosition = startPositionList[sort_index+1]
              if (currentStartPosition <= dragPosition && dragPosition < nextStartPosition){
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
          let cellPosition = startPositionList[sort]
          
          let isDraggedFront = dragPosition < startPosition 
          if (isDraggedFront) { 
            // 사이즈가 2면 cellPosition >= position 가 아니라 cellPosition+1 >= position이 되어야한다
            if (cellPosition <= startPosition && cellPosition + size -1 >= dragPosition) {
              
              return {
                ...item,
                sort: sort + 1,
              }
            }
          }
          let isDraggedBack = dragPosition > startPosition  
          if (isDraggedBack) {
            if (cellPosition > startPosition && cellPosition <= dragPosition) {
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
      // sort 순서대로 다시 정렬을 해줘야 위치 계산에서 오류가 안생긴다 
      // ex) getStartPositionList
      this.list.sort(function(a,b){
        return Number(a.sort) > Number(b.sort) ? 1: -1
      })

    },
    nomalizePosition(position, startPositionList){
      position = Math.max(position, -1)
      /*
        If you remove this line you can drag items to positions that
        are further than items array length
      */
      position = Math.min(position, startPositionList[this.list.length - 1])
      return position
    },
    getStartPositionList(){
      let {cellCountPerRow, list} = this
      let startPositionList = []
      let position = 0
      
      for (let i =0; i < list.length; i++){
        let size = list[i].size
        let startCellRowNumber = Math.floor(position / cellCountPerRow)
        let endCellRowNumber = Math.floor((position + size - 1) / cellCountPerRow)
        let isCellExceedRow = startCellRowNumber != endCellRowNumber;
        let emptyCellSize = isCellExceedRow ? cellCountPerRow - (position % cellCountPerRow): 0;
        let addedCellSize = emptyCellSize + size

        startPositionList.push(position)
        position += addedCellSize
      }

      return startPositionList
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
