<template>
  <div id="app">
    <div style="height:1000px;width:1600px;display:flex;align-items:stretch">
      <FeatureContour @exportPointsByMouseOver="handleExportPointsByMouseOver" @exportPointsByMouseLeave="handleExportPointsByMouseLeave" @exportLabels="handleExportLabels" ref="FeatureContour" style="flex:1.5 1.5 0;margin-right:10px;"/>
      <FeatureHistogram @selectedFeature="handleSelectFeature" ref="FeatureHistogram" style="flex:1 1 0;"/>
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3'
import FeatureContour from "./components/FeatureContour.vue"
import FeatureHistogram from "./components/FeatureHistogram.vue"

export default {
  name: 'App',
  components: {
    FeatureContour,
    FeatureHistogram,
  },
  methods:{
    handleSelectFeature(selectFeature){
      this.$refs['FeatureContour'].setFeatures(selectFeature);
      this.$refs['FeatureContour'].fetchContour();
    },
    handleExportLabels(exportLabels){
      this.$refs['FeatureHistogram'].updateLabel(exportLabels)
    },
    handleExportPointsByMouseOver(id){
      this.$refs['FeatureHistogram'].highlightBarById(id);
    },
    handleExportPointsByMouseLeave(id){
      this.$refs['FeatureHistogram'].unitHeightBarById(id)
    }
  },
  mounted(){
    let data = []
    d3.csv('static/特征等高线组件模拟数据.csv',(res)=>{
      //整理数据格式
      for(let key in res){
        if(key == 'id' || key =='label'){//id和label转为字符串
          res[key] = String(res[key])
        }
        else{
          res[key] = parseFloat(res[key])
        }
      }

      //装填数据
      data.push(res)

    }).then(()=>{
      this.$refs['FeatureHistogram'].start(data);
      this.$refs['FeatureContour'].setData(data);
    })
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
