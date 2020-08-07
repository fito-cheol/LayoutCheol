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
        <!-- https://stackoverflow.com/questions/41748934/how-to-create-dynamic-tag-based-on-props-with-vue-2 -->        
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
        {content:'c',size:2},{content:'d',size:1},
        {content:'e',size:1},
      ]
    }
  }
}
</script>

