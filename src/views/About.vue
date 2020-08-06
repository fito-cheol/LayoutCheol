<template>
  <div class="about">
    <layout
      :draggable="true"
      :sortable="true"
      :items="newItems"
      :cellWidth="width"
      :cellHeight="height"
      >
      <template slot="cell" scope="props">
          <button style="width:100%; height:100%">{{props.item}} - {{props.index}} / {{props.sort}} </button>
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
      width:100,
      height:100,
      items: [
        'a',
        'b',
        'c',
      ],
      newItems:[
        {content:'a', size:1},{content:'b',size:2},
        {content:'c',size:3},{content:'d',size:5},
        {content:'e',size:5},
      ]
    }
  }
}
</script>

