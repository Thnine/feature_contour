<template>
    <div class="feature-contour-container">
        <div class="feature-contour-title">
            <span>投影视图</span>
        </div>
        <div class="feature-contour-body">
            <div class="feature-contour-header">
                <div class="feature-contour-controller">
                    <div style="margin-left:30px;">
                        <b>等高线：</b>
                        <el-select
                            size="mini"
                            style="width:120px;"
                            v-model="selectedFeature">
                            <el-option       
                                label="无"
                                value="无">
                            </el-option>
                            <el-option       
                                v-for="feature in allFeaturesList"
                                :key="feature"
                                :label="feature"
                                :value="feature">
                            </el-option>
                        </el-select>
                    </div>
                    <div style="margin-left:30px;display:flex;align-items:center">
                        <b>颜色：</b>
                        <el-color-picker
                            style="margin-left:1px;"
                            v-model="contourColor"
                            @change="handleContourColorChange"
                            ></el-color-picker>
                    </div>
                    <svg ref="feature-contour-shape-legend" class="feature-contour-shape-legend"></svg>
                </div>
            </div>
            <div class="feature-contour-main">
                <div class="feature-contour-canvas-container">
                    <el-switch
                        v-model="dragMode"
                        @change="handleDragModeChange"
                        style="position:absolute;right:40px;top:10px;"
                        active-text="圈选"
                        active-value="圈选"
                        inactive-text="查看"
                        inactive-value="查看">
                    </el-switch>
                    <el-popover
                        v-model="annoBox_visible"
                        placement="bottom">
                        <div>
                            <div style="display:flex;align-items:center">
                                <b>标注为：</b>
                                <el-input v-model="anno_type" style="width:120px" size="mini"></el-input>
                            </div>
                            <div style="display:flex;margin-top:15px;align-items:center;justify-content:space-around">
                                <el-button type="primary" @click="handleEnterAnno" size="mini">确定</el-button>
                                <el-button type="danger" @click="handleCancelAnno" size="mini">取消</el-button>
                            </div>
                        </div>
                        <el-button 
                            icon="el-icon-edit"
                            type="primary"
                            slot="reference"
                            style="position:absolute;left:30px;top:20px;"
                            circle>
                        </el-button>                    
                    </el-popover>
                    <el-popconfirm
                        title="确定清除所有标注吗？"
                        @confirm="handleClearAnno">
                        <el-button 
                            slot="reference"
                            icon="el-icon-delete-solid"
                            type="danger"
                            style="position:absolute;left:30px;top:70px;margin-left:0px;"
                            circle ></el-button>
                    </el-popconfirm>

                    <svg ref="feature-contour-canvas" class="feature-contour-canvas"></svg>
                </div>
                <div class="feature-contour-color-legend-container">
                    <svg class="feature-contour-color-legend"></svg>
                </div>
            </div>
            <div class="feature-contour-foot">

            </div>

        </div>
        <!--悬浮信息版-->
        <InfoPanel :style="InfoPanelStyle" v-show="InfoPanelVisible" ref="infoPanel"/>

    </div>
</template>

<script>

import './FeatureContour.css'
import Vue from 'vue'
import axios from "axios"
import * as d3 from 'd3'
import lasso from "./d3-lasso";
import InfoPanel from './InfoPanel.vue'

import { Select,Option,ColorPicker,Message,Switch,Popover,Input,Popconfirm} from 'element-ui';



Vue.component(Select.name,Select)
Vue.component(Option.name,Option)
Vue.component(ColorPicker.name,ColorPicker)
Vue.component(Switch.name,Switch)
Vue.component(Input.name,Input)
Vue.component(Message.name,Message)
Vue.component(Popover.name,Popover)
Vue.component(Popconfirm.name,Popconfirm)

export default {
    name:'FeatureContour',
    components:{InfoPanel},
    data(){
        return {
            /**
             * 关键数据项
             */
            raw_data:[],//原始数据
            dimData:{},//实际数据


            /**
             * 配置数据
             */
            circleSize:100,
            rectSize:100,
            triangleSize:100,

            /**
             * 其他数据
             */
            colors:[],//备选颜色数组
            dragMode:'查看',//拖动模式
            lasso:null,//套索类
            zoom:null,//zoom类
            idAssign:[],//id序列
            chosenIndex:[],//被选择点的index（不是id）
            InfoPanelVisible:false,//悬浮板是否显示
            InfoPanelStyle:{
                top:'0px',
                left:'0px'
            },//信息版的动态style

            //特征相关
            allFeaturesList:[],//特征表
            usedFeaturesList:[],//实际被使用的特征列表
            selectedFeature:'无',//被选择的特征

            //类型相关
            labels:{},//label的相关属性 {'label1':{'color':'color1',...},...}
            labelAssign:[],//每个点被分配的标签类型
            annoFlag:[],//每个点是否为标注点
            annoBox_visible:false,//标注框的显示
            anno_type:'',//标注的事件类型

            //等高线相关
            contourColor:"#03a9f4",//等高线颜色
            contourColorScheme:undefined,//等高线渐变颜色
        }

    },


    methods:{

        /**
         * 外部接口
         */

        setData(raw_data){//传入绘图数据
            const self = this;
            self.raw_data = raw_data;
            /**
             * 初始化一些量
             */
            //解析features
            let tempFeatures = Object.keys(self.raw_data[0])
            tempFeatures.splice(tempFeatures.indexOf('id'),1)
            tempFeatures.splice(tempFeatures.indexOf('label'),1)
            self.allFeaturesList = tempFeatures;
            self.usedFeaturesList = JSON.parse(JSON.stringify(tempFeatures))
            //解析id
            self.idAssign.length = 0;
            self.idAssign = self.raw_data.map(v=>v.id)
            //解析label并分配动态颜色
            self.labels = {}
            self.labelAssign.length = 0;
            self.labelAssign = self.raw_data.map(v=>v.label)
            let labelSet = Array.from(new Set(self.labelAssign));
            for(let i = 0;i < labelSet.length;i++){
                if(labelSet[i] == ''){
                    self.labels[''] = {
                        'color':'#000000'
                    }
                }             
                else{
                    self.labels[labelSet[i]] = {
                        'color':self.colors[i]
                    }
                }
            }
            //selectedFeature
            self.selectedFeature = '无'
            //contourColor
            self.contourColor = '#03a9f4'
            //contourColorScheme
            self.contourColorScheme = undefined;
            //lasso
            self.lasso = null;
            //zoom
            self.zoom = null;
            //annoFlag
            self.annoFlag.length = 0;
            //anno_type
            self.anno_type = '';
            //drag_mode
            self.dragMode = '查看';
            //chosenIndex
            self.chosenIndex.length = 0;

            //绘制colorLegend
            self.drawColorLegend();
        },

        setFeatures(passedFeatures){//设置实际被使用的标签
            const self = this;
            self.usedFeaturesList.length = 0;
            for(let feature of passedFeatures){
                if(self.allFeaturesList.indexOf(feature) != -1){
                    self.usedFeaturesList.push(feature)
                }
            }
        },

        async fetchContour(){//获取投影和等高线
            const self = this;

            /**
             * 合法性检查
             */
            if(self.usedFeaturesList.length < 3){
                Message({
                    message: '特征数过少，请选择三个及以上的特征',
                    type: 'error'
                })
                return ;
            }



            /**
             * 请求
             */
            let allMatrix = []
            for(let i = 0;i < self.raw_data.length;i++){
                let row = []
                for(let feature of self.allFeaturesList){
                    row.push(self.raw_data[i][feature])
                }
                allMatrix.push(row)
            }
            let usedMatrix = []
            for(let i = 0;i < self.raw_data.length;i++){
                let row = []
                for(let feature of self.usedFeaturesList){
                    row.push(self.raw_data[i][feature])
                }
                usedMatrix.push(row)
            }
            let perturbationIndex = self.allFeaturesList.indexOf(self.selectedFeature)


            await axios({
              url:'requestContour',
              method:"POST",
              data:{
                'allMatrix':allMatrix,
                'usedMatrix':usedMatrix,
                'perturbationIndex':perturbationIndex,
              }
            }).then((res)=>{
                self.dimData = res.data
                console.log('dimData:',self.dimData)
            })

            /**
             * 初始化一些值
             */
            self.chosenIndex.length = 0;


            /**
             * 绘图
             */
            self.draw();
        },


        /**
         * 内部函数
         */
        draw(){//使用现有绘图数据
            const self = this;

            const svg = d3.select(self.$refs['feature-contour-canvas'])
            svg.selectAll("*").remove();


            // 设定架构
            const zoomDriver = svg.append('g').classed('feature-contour-zoom-driver',true)//由于接受zoom事件，防止svg监听的重用问题
            zoomDriver.append('rect').attr('width','100%').attr('height','100%').attr('fill','white')//支撑ZoomDriver的大小
            const zoomPlot = zoomDriver.append('g').classed('feature-contour-zoom-plot',true)
            const contourPlot = zoomPlot.append('g').classed('feature-contour-contour-plot',true)
            const scatterPlot = zoomPlot.append('g').classed('feature-contour-scatter-plot',true)

            // 计算尺寸
            const width = svg.node().getBoundingClientRect().width
            const height = svg.node().getBoundingClientRect().height;
            
            const size = (width + height) / 2;

            //准备数据
            let points = []
            for(let i = 0;i < self.dimData.points.length;i++){
                points.push({
                    'x':self.dimData.points[i][0],
                    'y':self.dimData.points[i][1],
                    'id':self.idAssign[i],
                })   
            }

            /**
             * 绘制的等高线
             */

            let scalarField = self.dimData.scalarField;

            let values = scalarField
            let n = (scalarField == undefined)?34:Math.sqrt(values.length);

            if(scalarField != undefined){
                var maxVal = Math.max.apply(Math, values);
                var minVal = Math.min.apply(Math, values);
                var numLines = 10;

                function scaleContour(contours, scale) {
                    return contours.map(({type, value, coordinates}) => (
                        {type, value, coordinates: coordinates.map(rings => (
                            rings.map(points => (
                            points.map(([x, y]) => ([
                                x*scale, y*scale
                            ]))
                            ))
                        ))}
                    ));
                }

                // 阈值从小到大，线性的设置10个，计算十次marching squares
                var contours = d3.contours()
                    .size([n, n])
                    .thresholds(d3.range(0, numLines).map(p => minVal + (maxVal - minVal) * (p + 1) / numLines))
                    (values);

                // 设置不同阈值的颜色，阈值越高则颜色越深
                self.contourColorScheme = d3.scaleLinear().range(['#ffffff',self.contourColor])
                    .domain([minVal, maxVal * 2]);
                    
                // 绘制等高线
                contourPlot.selectAll("path")
                    .data(scaleContour(contours,1))
                    .join('path')
                    .classed('contourPlotPath',true)
                    .attr("d", d3.geoPath(d3.geoIdentity().scale(size / n)))
                    .attr("fill", function (d) {return self.contourColorScheme(d.value) }) 


            }
            

            /**
             * 绘制降维点
             */

            const xScale = d3.scaleLinear().range([width / n * 2, width - width / n * 2]).domain(d3.extent(self.dimData.points, function (d) { return d[0]; }))
            const yScale = d3.scaleLinear().range([height - height / n * 2, height / n * 2]).domain(d3.extent(self.dimData.points, function (d) { return d[1]; }))
            scatterPlot.selectAll('*')
                       .data(points)
                       .join('path')
                       .classed('feature-contour-point',true)
                       .attr('d',(d,i)=>{
                            if(self.labelAssign[i] == ''){//类型未确定
                                return d3.symbol().type(d3.symbolSquare).size(self.circleSize)()
                            }
                            else if(self.annoFlag[i]){//标注
                                return d3.symbol().type(d3.symbolTriangle).size(self.triangleSize)()
                            }
                            else{
                                return d3.symbol().type(d3.symbolCircle).size(self.circleSize)();//固有类型
                            }

                       })
                       .style('transform',(d,i)=>{
                            return `translate(${xScale(d['x'])}px,${yScale(d['y'])}px)`
                       })
                       .attr('fill',(d,i)=>{
                            return self.labels[self.labelAssign[i]].color;
                       })
                       .classed('feature-contour-chosen',(d,i)=>{
                            if(self.chosenIndex.indexOf(i) != -1){
                                return true
                            }
                            return false;
                       })
                       .on('mousemove',function(d){//更新和显示信息板
                            self.InfoPanelVisible = true;//显示
                            self.$refs['infoPanel'].setMessageData({'id':d.id});//更新信息
                            /**
                            * 更新位置
                            */

                            self.InfoPanelStyle['top'] = `${d3.event.clientY + 10}px`
                            self.InfoPanelStyle['left'] = `${d3.event.clientX + 10}px`
                            /**
                             * 悬停导出事件
                             */
                            self.exportPointsByMouseOver(d.id)
                        })
                        .on('mouseleave',(d)=>{//隐藏信息板
                            this.InfoPanelVisible = false;//隐藏
                            /**
                             * 离开导出事件
                             */
                            self.exportPointsByMouseLeave(d.id)
                        })

            /**
             * 设定lasso事件
             */
            let lasso_start = () => {
                //清空所有选择点
                self.chosenIndex.length = 0;
                self.lasso.items().classed('feature-contour-chosen',false);
            };
            let lasso_draw = () => {
                self.lasso.possibleItems().classed('feature-contour-chosen',(d,i)=>{
                    if(self.labelAssign[self.idAssign.indexOf(d.id)] == ''){//只有事件类型未确定的点才能被选中
                        return true;
                    }
                    return false;
                })
                self.lasso.notPossibleItems().classed('feature-contour-chosen',false) 
            
            };
            let lasso_end = () => {
                self.lasso.selectedItems()
                          .classed('feature-contour-chosen',(d,i)=>{
                                if(self.labelAssign[self.idAssign.indexOf(d.id)] == ''){//只有事件类型未确定的点才能被选中
                                    return true;
                                }
                                return false;
                           })
                           .each((d,i)=>{
                                if(self.labelAssign[self.idAssign.indexOf(d.id)] == ''){//只有事件类型未确定的点才能被选中
                                    self.chosenIndex.push(self.idAssign.indexOf(d.id))
                                }
                           })
            };
            self.lasso = lasso()
                .closePathSelect(true)
                .closePathDistance(100)
                .items(svg.selectAll('.feature-contour-point'))
                .targetArea(svg)
                .on("start", lasso_start)
                .on("draw", lasso_draw)
                .on("end", lasso_end);
            self.lasso.closePathDistance(2000);

            /**
             * 设定zoom事件
             */
            
            
            self.zoom = d3.zoom()
                       .scaleExtent([0.1, 40])
                       .translateExtent([[-10000, -10000], [10000, 10000]])
                       .on("zoom",()=>{
                            zoomPlot.attr("transform",d3.event.transform)
                        })

            
            /**
             * 绑定事件
             */
            if(self.dragMode=='圈选'){
                svg.on('.drag',null);
                zoomDriver.on('.zoom',null)
                svg.call(self.lasso)
            }
            else if(self.dragMode=='查看'){
                svg.on('.drag',null);
                zoomDriver.on('.zoom',null)
                zoomDriver.call(self.zoom)
            }
            
            


            // //缩放
            // let AllScale = 1.01
            // svg.attr('transform',`scale(${AllScale})`)

        },

        drawShapeLegend(){//绘制图形的形状的legend
            const self = this;
            const shapeLegend = d3.select(this.$refs['feature-contour-shape-legend']);
            shapeLegend.selectAll('*').remove();
            
            const certainSymbol = d3.symbol().type(d3.symbolCircle).size(self.circleSize)
            const uncertainSymbol = d3.symbol().type(d3.symbolSquare).size(self.rectSize)
            const annoSymbol = d3.symbol().type(d3.symbolTriangle).size(self.triangleSize)
            
            //certainLegend
            const certainLegend = shapeLegend.append('g');
            certainLegend.append('path')
                         .attr("d",certainSymbol())
                         .attr('transform','translate(0,-12)')
            certainLegend.append('text')
                         .attr('text-anchor','middle')
                         .attr('transform','translate(0,12)')
                         .text('类型确定')   
            certainLegend.style('transform','translate(20%,50%)') 

            //certainLegend
            const uncertainLegend = shapeLegend.append('g');
            uncertainLegend.append('path')
                         .attr("d",uncertainSymbol())
                         .attr('transform','translate(0,-12)')
            uncertainLegend.append('text')
                         .attr('text-anchor','middle')
                         .attr('transform','translate(0,12)')
                         .text('类型未确定')   
            uncertainLegend.style('transform','translate(50%,50%)')     

            //annoLegend
            const annoLegend = shapeLegend.append('g');
            annoLegend.append('path')
                         .attr("d",annoSymbol())
                         .attr('transform','translate(0,-12)')
            annoLegend.append('text')
                         .attr('text-anchor','middle')
                         .attr('transform','translate(0,12)')
                         .text('人为标记类型')   
            annoLegend.style('transform','translate(80%,50%)')     


            // shapeLegend.append('circle')
            //     .attr('cx',70)
            //     .attr('cy',20)
            //     .attr('r',self.circleSize)
            //     .attr('fill','black')
            //     .attr('stroke','black')
            //     .attr('stroke-width',2)
            // shapeLegend.append('text')
            //     .attr('x',30)
            //     .attr('y',45)
            //     .text('事件类型确定')
            // shapeLegend.append('rect')
            //     .attr('x',200)
            //     .attr('y',15)
            //     .attr('height',self.rectSize)
            //     .attr('width',self.rectSize)
            //     .attr('fill','black')
            //     .attr('stroke','black')
            //     .attr('stroke-width',2)
            // shapeLegend.append('text')
            //     .attr('x',155)
            //     .attr('y',45)
            //     .text('事件类型未确定')
            // shapeLegend.append("polygon")
            //     .attr("points",()=>{
            //         let X = 330
            //         let Y = 22
            //         let p1_X = X;
            //         let p1_Y = Y - self.triangleSize * Math.sqrt(3)  / 3;
            //         let p2_X = X - 0.5 * self.triangleSize;
            //         let p2_Y = Y + self.triangleSize * Math.sqrt(3) / 2 * 1 / 3;
            //         let p3_X = X + 0.5 * self.triangleSize;
            //         let p3_Y = Y + self.triangleSize * Math.sqrt(3) / 2 * 1 / 3;                
            //         return p1_X + "," + p1_Y + " " + p2_X + "," + p2_Y + " " + p3_X + "," + p3_Y; 
            //     })
            //     .attr('fill','black')
            //     .attr('stroke','black')
            //     .attr('stroke-width',2)
            // shapeLegend.append('text')
            //     .attr('x',272)
            //     .attr('y',45)
            //     .text('人为标记事件类型')
        },

        drawColorLegend(){//绘制图像的颜色的legend
            const svg = d3.select('.feature-contour-color-legend')
            const self = this;
            const padding = {//legend的padding
                'left':10,
                'right':10,
                'top':10,
                'bottom':10,
            }

            //清空画布
            svg.selectAll('*').remove();

            //处理异常
            if(self.labels === undefined || self.labels === null || Object.keys(self.labels).length == 0)
                return;

            const unitWidth = 80;
            const unitHeight = 40;
            const textSize = 22;
            const height = svg.node().getBoundingClientRect().height;

            //包装this.groups为array

            const legendPlot = svg.append('g')


            //计算一列能有多少个legendUnit
            let columnMax = Math.floor((height - padding.top - padding.bottom)/unitHeight);
            const legendUnit = legendPlot.selectAll('*')
                .data(Object.keys(self.labels))
                .join('g')
                .attr('transform',function(d,i){
                    return `translate(${Math.floor(i / columnMax) * unitWidth + padding.left},${(i % columnMax) * unitHeight + padding.top})`
                })
            
            //legend rect
            legendUnit.append('circle')
                .attr('cx',0.3 * unitWidth)
                .attr('cy',0.5 * unitHeight)
                .attr('r',8)
                .attr("fill",d=>self.labels[d].color)

            //legend text
            legendUnit.append('text')
                .text(d=>{
                    if(d == '')return '未确定'
                    else return d
                })
                .attr('font-size',textSize)
                .attr('x',d=>0.5 * unitWidth)
                .attr('y',function(d){
                    return 0.5 * unitHeight + 0.4 * textSize;
                })
            
            svg.style('width',unitWidth * Math.ceil(self.labels.length / columnMax)  + padding.right + padding.left)

        },

        updateScatter(){//更新散点的视图状态
            const self = this;
            const svg = d3.select(self.$refs['feature-contour-canvas'])
            const scatter = svg.selectAll('.feature-contour-point')
            scatter.attr('d',(d,i)=>{
                            if(self.labelAssign[i] == ''){//类型未确定
                                return d3.symbol().type(d3.symbolSquare).size(self.circleSize)()
                            }
                            else if(self.annoFlag[i]){//标注
                                return d3.symbol().type(d3.symbolTriangle).size(self.triangleSize)()
                            }
                            else{
                                return d3.symbol().type(d3.symbolCircle).size(self.circleSize)();//固有类型
                            }
                    })
                    .attr('fill',(d,i)=>{
                        return self.labels[self.labelAssign[i]].color;
                    })
                    .classed('feature-contour-chosen',(d,i)=>{
                            if(self.chosenIndex.indexOf(i) != -1){
                                return true
                            }
                            return false;
                    })


        },


        handleContourColorChange(){
            const self = this;
            
            const contourPlotPath = d3.selectAll('.contourPlotPath');
            self.contourColorScheme.range(['#ffffff',self.contourColor])
            contourPlotPath.attr("fill", function (d) {return self.contourColorScheme(d.value) }) 

        },

        handleDragModeChange(){
            const self = this;
            const svg = d3.select(self.$refs['feature-contour-canvas'])
            const zoomDriver = svg.select('.feature-contour-zoom-driver')
            if(self.dragMode=='圈选'){
                svg.on('.drag',null);
                zoomDriver.on('.zoom',null)
                svg.call(self.lasso)
            }
            else if(self.dragMode=='查看'){
                svg.on('.drag',null);
                zoomDriver.on('.zoom',null)
                zoomDriver.call(self.zoom)
            }  
        },

        handleEnterAnno(){//确认标注
            const self = this;

            if(self.chosenIndex.length == 0){
                Message({
                    message: '请先选择点再进行标注',
                    type: 'error'
                })
            }
            else if(self.anno_type == ''){
                Message({
                    message: '请输入合法的类型名',
                    type: 'error'
                })
            }
            else{//通过合法性判断

                if(Object.keys(self.labels).indexOf(self.anno_type) != -1){//如果是已有的类

                }
                else{//新类型
                    //分配新颜色
                    self.labels[self.anno_type] = {'color':self.colors[Object.keys(self.labels).length]}
                }
                for(let index of self.chosenIndex){//分配类型
                        self.labelAssign[index] = self.anno_type;
                        self.annoFlag[index] = true;
                    }
                //清空选择
                self.chosenIndex.length = 0;
                self.updateScatter();
                self.drawColorLegend();
                self.anno_type = ''
                self.annoBox_visible = false;
                Message({
                    message: '标注成功',
                    type: 'success'
                })
                self.exportLabels();
            }

        },

        handleCancelAnno(){//取消标注
            const self = this;
            self.annoBox_visible = false
        },

        handleClearAnno(){//清除标注
            const self = this;
            for(let i = 0;i < self.annoFlag.length;i++){
                if(self.annoFlag[i]){
                    self.labelAssign[i] = ''
                }
                self.annoFlag[i] = false;
            }
            self.updateScatter();
            Message({
                    message: '已清除标注',
                    type: 'success'
                })
            self.exportLabels();
        },
        
        exportLabels(){//导出标注结果
            const self = this;
            //导出标注结果
            let exportValue = {}
            for(let i = 0;i < self.idAssign.length;i++){
                exportValue[self.idAssign[i]] = self.labelAssign[i];
            }
            self.$emit('exportLabels',exportValue)
        },
        exportPointsByMouseOver(id){
            const self = this;
            self.$emit('exportPointsByMouseOver',id)
        },
        exportPointsByMouseLeave(id){
            const self = this;
            self.$emit('exportPointsByMouseLeave',id)

        }
    },


    mounted(){
        //生成备选颜色
        this.colors = [...d3.schemeSet1,...d3.schemeSet2,...d3.schemeSet3,...d3.schemeTableau10]
        //绘制legend
        this.drawShapeLegend();
    }
}
</script>

<style scoped>


</style>