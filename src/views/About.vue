<template>
  <div class="about">
    <layout
      :draggable="true"
      :sortable="true"
      :items="newItems"
      cellWidth=100
      cellHeight=100
      
      >
      <template slot="cell" scope="props">
          <button style="width:80px; height:80px">{{props.item}} - {{props.index}} / {{props.sort}} </button>
      </template>
    </layout>
  </div>
</template>

<script>
// @ is an alias to /src
import Layout from '@/components/GridCustom/Layout.vue'

export default {
  name: 'Home',
  components: {
    Layout
  },
  data () {
    return {
      items: [
        'a',
        'b',
        'c',
      ],
      newItems:[
        {
            content:'a',
            size:1
        },
        {
            content:'b',
            size:1
        },
        {
            content:'C',
            size:2
        },
        {
            content:'D',
            size:1
        }
      ]
    }
  }
}
</script>

