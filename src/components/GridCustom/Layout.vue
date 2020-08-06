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
              :row-count="rowCount"
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
    center: {
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
      return Math.ceil(this.list.length / this.rowCount) *
        this.cellHeight
    },

    style () {
      return {
        height: this.height + 'px'
      }
    },

    rowCount () {
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
      // console.log('sortList', itemIndex, gridPosition, this.list,)
      let targetItem = this.list.find(item => item.index === itemIndex)
      let targetItemSort = targetItem.sort
      let startPointList = this.getStartPointList()
      // console.log(startPointList)
      let startPoint = startPointList[targetItemSort]


      // gridPosition를 0과 list.length 사이에 위치시킨다
      /*
        Normalizing new grid position
      */
      gridPosition = Math.max(gridPosition, -1)
      /*
        If you remove this line you can drag items to positions that
        are further than items array length
      */
      gridPosition = Math.min(gridPosition, startPointList[this.list.length - 1])

      
      // sort 의 값만 바꿔준다
      
      let isDragChangePosition =  startPoint !== gridPosition
      if (isDragChangePosition) {
        console.log("-------", startPoint, gridPosition, targetItemSort)
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
              // FIXME: 여기 조건이 마지막 sort문을 못넘김 
              if (Number(sort_index) == startPointList.length -1 ){
                new_sort = Number(sort_index)
                break;
              }
              if (startPointList[sort_index] <= gridPosition && gridPosition< startPointList[Number(sort_index)+1] ){
                new_sort = Number(sort_index)
                break;
              }      
            }
            console.log("잘 나와야 하는데", new_sort)
            return {
                  ...item,
                  sort: new_sort,
                }
          }

          const { sort } = item
          let comparedStartPoint = startPointList[sort]
          
          let isDraggedLeft = gridPosition < startPoint 
          if (isDraggedLeft) { 
            if (comparedStartPoint <= startPoint && comparedStartPoint >= gridPosition) {
              console.log('+1 List', sort, comparedStartPoint, gridPosition, startPoint)
              
              return {
                ...item,
                sort: sort + 1,
              }
            }
          }
          let isDraggedRight = gridPosition > startPoint  
          if (isDraggedRight) {
            if (comparedStartPoint > startPoint && comparedStartPoint <= gridPosition) {
              console.log('-1 List', sort, comparedStartPoint, gridPosition, startPoint)
              // FIXME: 여기 여러번 들어가기 때문에 해결해줘야함
              // 문제1 startPoint가 안바뀜, 그게 문제
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
    getStartPointList(){
      let accumulationList = []
      let centerPointList = []
      let accumulSpace = 0
      let {rowCount, list} = this
      for (let i =0; i < list.length; i++){ //sort는 1부터 시작하니까 -1 해준다
        
        let size = list[i].size
        let rowNumber = Math.floor((accumulSpace + size - 1) / rowCount)
        let isOverFlow = Math.floor(accumulSpace / rowCount) != rowNumber;
        let emptySpace = isOverFlow ? rowCount - (accumulSpace % rowCount): 0;
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
