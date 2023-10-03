<template>
  <div>
    <div style="height:1000px;width:1600px;display:flex;align-items:stretch">
      <FeatureContour @exportPointsByMouseOver="handleExportPointsByMouseOver" @exportPointsByMouseLeave="handleExportPointsByMouseLeave" @exportLabels="handleExportLabels" ref="FeatureContour" style="flex:1.5 1.5 0;margin-right:10px;"/>
      <FeatureHistogram @selectedFeature="handleSelectFeature" ref="FeatureHistogram" style="flex:1 1 0;"/>
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3'
import FeatureContour from "./FeatureContour.vue"
import FeatureHistogram from "./FeatureHistogram.vue"

export default {
  name: 'ComponentContainer',
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
    },
    start(data){
      this.$refs['FeatureHistogram'].start(data);
      this.$refs['FeatureContour'].setData(data);

    },
  },

}
</script>

<style>

</style>