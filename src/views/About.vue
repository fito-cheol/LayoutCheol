<template>
  <div class="about">
    <layout
      :draggable="true"
      :sortable="true"
      :items="items"
      :cellWidth="width"
      :cellHeight="height"
      >
      <template slot="cell" scope="props">
        <!-- https://stackoverflow.com/questions/41748934/how-to-create-dynamic-tag-based-on-props-with-vue-2 -->        
        <component :is="props.tag" v-model="props.content" style="overflow: hidden; margin:0px; width:100%; height:50%"> {{props.content}} </component>
        <button style="width:100%; height:50%">{{props.tag}} {{props.item}} - {{props.index}} / {{props.sort}} </button>
      </template>
    </layout>
  </div>
</template>


<script>
// @ is an alias to /src
import Layout from '@/components/GridCustom/Layout.vue'
import ChartCustom from '@/components/Chart/Custom.vue'
import ChartTest from '@/components/Chart/Test.vue'

export default {
  name: 'Home',
  components: {
    Layout,
    ChartCustom,
    ChartTest
  },
  data () {
    return {
      width:100,
      height:100,
      items:[
        {content:'a', size:1, tag:'h1'},{content:'b',size:2, tag:'h2'},
        {content:'c',size:2, tag:'h3'},{content:'d',size:1, tag:'chart-custom'},
        {content:'e',size:1, tag:'chart-test'},
      ]
    }
  }
}
</script>

