<template>
    <div class="feature-histogram-container">
        
        <div class="feature-histogram-title">
            <span>特征分布直方图</span>

        </div>
        <div class="feature-histogram-body">
            <div class="feature-histogram-header">

            </div>
            <div class="feature-histogram-main">
                <div ref="feature-histogram-canvas-container" class="feature-histogram-canvas-container">
                    <div style="position:absolute;display:flex;top:10px;left:10px">
                        <el-button
                            type="primary"
                            size="mini"
                            @click="EMDFeautreSort">
                            排序
                        </el-button>
                        <el-button
                            type="primary"
                            size="mini"
                            @click="handleFeatureSortReset">
                            还原
                        </el-button>
                    </div>
                    <svg ref="feature-histogram-canvas" class="feature-histogram-canvas"></svg>
                </div>
            </div>
            <div class="feature-histogram-foot">
                <div>
                    <el-pagination
                        small
                        layout="prev, pager, next, jumper"
                        :current-page.sync="currentPage"
                        :page-count="pageCount"
                        @current-change="handleCurrentPageChange">
                    </el-pagination> 
                </div>
                <div style="display:flex;justify-content:space-between">
                    <el-button type="primary" size="mini" @click="exportSelectedFeature">确定</el-button>
                    <el-button type="danger" size="mini" @click="handleResetSelectFeature">重置</el-button>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
import Vue, { getCurrentInstance } from 'vue'
import * as d3 from 'd3'
import { group,bin } from 'd3-array';
import {Button,Pagination} from 'element-ui'
import 'element-ui/lib/theme-chalk/index.css';
import './FeatureHistogram.css'
import rectImg from '../assets/rect.svg'
import tickImg from '../assets/tick.svg'
import axios from 'axios';

Vue.component(Button.name,Button)
Vue.component(Pagination.name,Pagination)


export default {
    name:'FeatureHistogram',
    data(){
        return {
            /**
             * 关键数据项
             */
            raw_data:[],//原始数据

            use_data:[],//使用数据（label可能变化）

            // draw_data:{
            //     'label1':{feature1:[v0,v1,...],...},
            //     ...
            // }

            draw_data:{},//绘图数据

            /**
             * 配置项
             */
            padding:{
                top:20,
                right:20,
                bottom:20,
                left:20,
            },
            labelAreaSize:40,//事件标签区域的长度
            featureAreaSize:120,//特征标签区域的长度（含按钮）
            unitHeight:150,//直方图单元的高度
            unitWidth:250,//直方图单元的宽度
            thresholdNum:10,//直方图均分的区间数目
            
            /**
             * 其他项
             */
            featureList:[],//特征名的list（顺序决定了显示的顺序）
            rawFeatureList:[],//原始顺序的特征名list
            labelList:[],//label的list（顺序决定了显示的顺序）
            showFeatureList:[],//展示的feature的list（顺序由featureList的顺序决定）
            pageCount:1,//页数
            currentPage:1,//当前页
            featureNumInAPage:0,//一页内容纳的特征数
            featureThresholds:{},//每个特征的thresholds
            selectedFeature:[], //被选择的feature的list（顺序无意义）

        }
    },
    methods:{

        /**
         * 外部接口
         */




        start(raw_data){//依照当前组件的原始数据进行重配置（恢复出厂状态）并绘图

            const self = this;

            self.setData(raw_data)

            //配置
            self.config();

            //重新生成绘图数据
            self.generateDrawData();

            //重绘图
            self.draw();
            
        },

        updateLabel(new_labels){//数据的label被修改
            /**
             * new_labels:{'id1':'label1','id2':'label2',...}
             */
            const self = this;
            
            //更新use_data
            for(let i = 0;i < self.use_data.length;i++){
                if(new_labels[self.use_data[i].id] !== undefined){
                    self.use_data[i].label = new_labels[self.use_data[i].id]
                }
            }
            
            //配置
            self.config();

            //重新生成绘图数据
            self.generateDrawData();

            //重绘图
            self.draw();

        },








        /**
         * 
         * 内部函数
         *  
         */

         setData(raw_data){//传入原始数据

            const self = this;

            self.raw_data = raw_data;
            self.use_data = JSON.parse(JSON.stringify(raw_data))
            console.log(self.use_data)

            self.init();

        },


        init(){//初始化，只会再源数据被修改时执行一次
            const self = this;

            self.featureList.length = 0;
            self.rawFeatureList.length = 0;
            self.labelList.length = 0;
            self.showFeatureList.length = 0;
            self.selectedFeature.length = 0;
            self.featureNumInAPage = 0;
            self.pageCount = 1;
            self.currentPage = 1;
            self.draw_data = {};
            self.featureThresholds = {};
    
            //计算特征列表（初始顺序）
            let tempFeatures = Object.keys(self.use_data[0])
            tempFeatures.splice(tempFeatures.indexOf('id'),1)
            tempFeatures.splice(tempFeatures.indexOf('label'),1)
            self.featureList = tempFeatures;
            self.rawFeatureList = JSON.parse(JSON.stringify(tempFeatures))


        },

        config(){//根据use_data生成配置数据，可以执行多次
            /**
             * 初始化数据
             */
           
            const self =  this;

            const height = self.$refs['feature-histogram-canvas-container'].getBoundingClientRect().height;

            
            //统计标签
            self.labelList = Array.from(new Set(self.use_data.map(v=>v.label)))
            if(self.labelList.indexOf('') != -1){//把未确定的标签排到最后
                self.labelList.splice(self.labelList.indexOf(''),1)
                self.labelList.push('')
            }

            //计算组件在当前页面能容纳的特征数目
            self.featureNumInAPage = Math.floor(1.0 * (height - self.padding.top - self.padding.bottom - self.labelAreaSize) / this.unitHeight);
            self.pageCount = Math.ceil(1.0 * self.featureList.length / self.featureNumInAPage);

        },

        generateDrawData(){//依照现有配置生成绘图数据(showFeatures,featureThresholds,draw_data)

            const self = this;

            //生成showFeatures
            self.showFeatureList.length = 0;
            for(let i = self.featureNumInAPage * (self.currentPage - 1);i < self.featureList.length && i < self.featureNumInAPage * self.currentPage;i++){
                self.showFeatureList.push(self.featureList[i]);
            }

            //计算featureThresholds
            self.featureThresholds = {};
            for(let feature of self.featureList){

                console.log(d3.extent(self.use_data,v=>v[feature]))

                let featureBin = bin().thresholds(self.thresholdNum)(self.use_data.map(v=>v[feature]))

                self.featureThresholds[feature] = []
                self.featureThresholds[feature].push(featureBin[0].x0)

                for(let i = 0;i < featureBin.length;i++){
                    self.featureThresholds[feature].push(featureBin[i].x1)
                }

                console.log(self.featureThresholds[feature])
                
            }

            //生成draw_data
            self.draw_data = {}
            group(self.use_data,d=>d.label).forEach((element,label) => {
                

                self.draw_data[label] = {}
                
                //按特征装填数据

                for(let feature of self.featureList){
                    //对每个绘图unit的数据进行分统
                    const unitBins = bin()
                                    .domain([self.featureThresholds[feature][0],self.featureThresholds[feature][self.featureThresholds[feature].length-1]])
                                    .thresholds(self.featureThresholds[feature].slice(1,-1))
                                    (element.map(v=>v[feature]))
                    
                    const unitBinRadio = []
                    for(let unitBin of unitBins){
                        unitBinRadio.push(1.0 * unitBin.length / element.length)
                    }
                    
                    self.draw_data[label][feature] = unitBinRadio

                }

                

            });
        },

        draw(){//依据已有的绘图数据和配置项画图
            /**
             * 
             * 依据已有的数据和配置项画图
             * 这部分应当只负责画图，不应该生成、修改数据和配置项
             * 
             */

            const self = this;
            const svg = d3.select(self.$refs['feature-histogram-canvas']);

            svg.selectAll('*').remove();

            const width = self.unitWidth * self.labelList.length + self.featureAreaSize + self.padding.left + self.padding.right;
            const height = self.$refs['feature-histogram-canvas'].getBoundingClientRect().height;
            svg.attr('width',width)

            console.log('draw_data:',self.draw_data)


            //绘制事件标签
            const labelPlot = svg.append('g'); 
            labelPlot
                .selectAll('*')
                .data(self.labelList)
                .join('text')
                .text(d=>{
                    if(d == '') return '未确定'
                    else return d;
                })
                .attr('text-anchor','middle')
                .attr('dominant-baseline','central')
                .style('font-size','18px')
                .attr('x',(d,i)=>{
                    return self.padding.left + self.featureAreaSize + i * self.unitWidth + 0.5 * this.unitWidth 
                })
                .attr('y',self.padding.top + 0.5 * self.labelAreaSize)
            
            //绘制特征标签
            const featurePlot = svg.append('g')
            const featureTextPlot = featurePlot.append('g')
            const featureCheckPlot = featurePlot.append('g')
            featureTextPlot//特征文本
                .selectAll('*')
                .data(self.showFeatureList)
                .join('text')
                .text(d=>d)
                .attr('dominant-baseline','central')
                .style('font-size','16px')
                .attr('x',self.padding.left + 30 + 10)
                .attr('y',(d,i)=>self.padding.top + self.labelAreaSize + i * self.unitHeight + 0.5 * self.unitHeight)
            featureCheckPlot//特征选择控件
                .selectAll('*')
                .data(self.showFeatureList)
                .join('image')
                .attr('width',30)
                .attr('height',30)
                .attr('x',self.padding.left + 5)
                .attr('y',(d,i)=>self.padding.top + self.labelAreaSize + i * self.unitHeight + 0.5 * self.unitHeight - 15)
                .style('cursor','pointer')
                .attr('href',(d)=>{
                    if(self.selectedFeature.indexOf(d) != -1){//处于被选中状态
                        return tickImg;
                    }
                    else{//处于未被选中状态
                        return rectImg;
                    }
                })
                .style('filter',(d,i)=>{
                    if(self.selectedFeature.indexOf(d) != -1){//处于被选中状态
                        return 'invert(39%) sepia(95%) saturate(1645%) hue-rotate(191deg) brightness(101%) contrast(102%)';
                    }
                    else{//处于未被选中状态
                        return null;
                    }   
                })
                .on('click',(d,i)=>{
                    if(self.selectedFeature.indexOf(d) != -1){//处于被选中状态，则取消选择
                        self.selectedFeature.splice(self.selectedFeature.indexOf(d),1)
                    }
                    else{//处于未被选中的状态，则选择
                        self.selectedFeature.push(d)
                    }
                    self.update();
                })


            //绘制直方图
            const matrix = svg.append('g')
            for(let i = 0;i < self.labelList.length;i++){
                let e = self.labelList[i];
                for(let j = 0;j < self.showFeatureList.length;j++){
                    let f = self.showFeatureList[j]       
                    let unitDrawData = self.draw_data[e][f]

                    const scaleX = d3.scaleLinear()
                                    .domain([self.featureThresholds[f][0],self.featureThresholds[f][self.featureThresholds[f].length-1]])
                                    .range([0,0.8*self.unitWidth])

                    const scaleY = d3.scaleLinear()
                                    .domain([1,0])
                                    .range([-0.72*self.unitHeight,0])

                    
                    let leftAxis = d3.axisLeft(scaleY).ticks(7)
                    let bottomAxis = d3.axisBottom(scaleX).tickValues(self.featureThresholds[f]).tickSize(8);
                    let unit = matrix.append('g')
                                     .attr('transform',`translate(${self.padding.left + self.featureAreaSize + i * self.unitWidth},${self.padding.top + self.labelAreaSize + j * self.unitHeight})`)
                    

                    let axisBiasX = 35
                    let axisBiasY = self.unitHeight - 26
                    unit.append('g')
                        .attr('transform',`translate(${axisBiasX-1},${axisBiasY})`)
                        .call(leftAxis)
                    const bottomAxisPlot =  unit.append('g')
                                                .attr('transform',`translate(${axisBiasX-1},${axisBiasY})`)
                                                .call(bottomAxis)
                    bottomAxisPlot.selectAll('text').style('font-size','9px')
                    unit.append('g')
                        .attr('transform',`translate(${axisBiasX},${axisBiasY})`)
                        .selectAll('rect')
                        .data(unitDrawData)
                        .enter()
                        .append('rect')
                        .attr('id',(d,i)=>'Histogrambar'+ e + f + i)
                        .classed('unHighlightBar',true)
                        .classed('HighlightBar',false)
                        .attr('x',(d,i)=>scaleX(self.featureThresholds[f][i]))
                        .attr('y',0)
                        .attr('width',(d,i)=>scaleX(self.featureThresholds[f][i+1])-scaleX(self.featureThresholds[f][i]))
                        .attr('height',(d,i)=>scaleY(-unitDrawData[i]))
                        .style('transform','rotateX(180deg)')
                    

                    unit.append('rect')
                        .attr('x',0)
                        .attr('y',0)
                        .attr("width",self.unitWidth)
                        .attr("height",self.unitHeight)
                        .attr("stroke-width", 2)
                        .attr("stroke", 'black')
                        .attr("fill-opacity",0)
                        .on('mouseover',function(){
                            d3.select(this.parentNode)
                                .append('text')
                                .text(`${self.labelList[i] == '' ? '未确定' : self.labelList[i]} : ${self.showFeatureList[j]}`)
                                .classed('matrix_tip',true)
                                .style('font-size','15px')
                                .attr('x',function(){
                                    return self.unitWidth - this.getBoundingClientRect().width - 7
                                })
                                .attr('y',20)

                        })
                        .on('mouseout',function(){
                            d3.selectAll('.matrix_tip')
                            .remove()
                        })

                }
            }
        },

        update(){//依照现有配置更新绘图数据，然后图
            this.generateDrawData();
            this.draw(); 
        },

        handleCurrentPageChange(){//currentPage被修改的回调函数
            this.update();
        },

        handleResetSelectFeature(){//重置被选择的特征
            this.selectedFeature.length = 0;
            this.update();
        },

        handleFeatureSortReset(){

            const self = this;

            self.featureList = JSON.parse(JSON.stringify(self.rawFeatureList));
            self.update();
        },

        exportSelectedFeature(){//导出被选择的feature

            this.$emit('selectedFeature',this.selectedFeature)
        },

        EMDFeautreSort(){//调用emd算法进行排序
            const self = this;

            axios({
              url:'requestFeatureSort',
              method:"POST",
              data:{
                'dataH':self.draw_data,
                'features':self.featureList,
                'labels':self.labelList,
                'ticks':self.featureThresholds,
              },
            }).then(res=>{
                self.featureList = res.data.map(v=>v[0])
                self.update();
            })

        },

        highlightBarById(id){//按照节点id点亮bar

            const self = this;
            const svg = d3.select(self.$refs['feature-histogram-canvas']);

            /**
             * 合法性检查
             */
            if(self.use_data.map(v=>v.id).indexOf(id) == -1)//如果输入的是不存在的id
                return ;
            //检索对应id的index
            let index = self.use_data.map(v=>v.id).indexOf(id);
            //获取对应点的label
            let label = self.use_data[index].label;
            //高亮点
            for(let f of self.showFeatureList){
                //计算顺序
                let threshold = self.featureThresholds[f];

                let binIndex = 0;
                for(let i = 0;i<threshold.length-1;i++){
                    if(self.use_data[index][f] >= threshold[i]){
                        binIndex = i;
                    }
                }
                let barId = 'Histogrambar'+ label + f + binIndex
                svg.select(`#${barId}`).classed('unHighlightBar',false)
                                       .classed('HighlightBar',true)

            }


        },

        unitHeightBarById(id){//按照节点id熄灭bar
            const self = this;
            const svg = d3.select(self.$refs['feature-histogram-canvas']);

            /**
             * 合法性检查
             */
            if(self.use_data.map(v=>v.id).indexOf(id) == -1)//如果输入的是不存在的id
                return ;
            //检索对应id的index
            let index = self.use_data.map(v=>v.id).indexOf(id);
            //获取对应点的label
            let label = self.use_data[index].label;
            //高亮点
            for(let f of self.showFeatureList){
                //计算顺序
                let threshold = self.featureThresholds[f];

                let binIndex = 0;
                for(let i = 0;i<threshold.length-1;i++){
                    if(self.use_data[index][f] >= threshold[i]){
                        binIndex = i;
                    }
                }
                let barId = 'Histogrambar'+ label + f + binIndex
                svg.select(`#${barId}`).classed('unHighlightBar',true)
                                       .classed('HighlightBar',false)

            }

        }

    },

}
</script>

<style scoped>

</style>