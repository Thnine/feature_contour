<template>
  <div id="app">
    <ComponentContainer ref="Component"/>
  </div>
</template>

<script>
import * as d3 from 'd3'
import ComponentContainer from "./components/ComponentContainer.vue"

export default {
  name: 'App',
  components: {
    ComponentContainer,
  },
  mounted(){
    let data = []
    // d3.csv('static/特征等高线组件模拟数据.csv',(res)=>{
    //   //整理数据格式
    //   for(let key in res){
    //     if(key == 'id' || key =='label'){//id和label转为字符串
    //       res[key] = String(res[key])
    //     }
    //     else{
    //       res[key] = parseFloat(res[key])
    //     }
    //   }

    //   //装填数据
    //   data.push(res)

    // }).then(()=>{
    //   console.log('data:',data)
    //   this.$refs['FeatureHistogram'].start(data);
    //   this.$refs['FeatureContour'].setData(data);
    // })
    d3.json('static/特征等高线组件模拟数据.json').then(res=>{
      data = res;
      console.log('data',data)
      this.$refs['Component'].start(data)
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
