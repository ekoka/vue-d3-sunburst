<template>
<div class="">
    <div class="graph">
    </div>
    <div style="min-height:50px">
        <label class="">{{hovered.name}}</label>
    </div>
</div>
</template>

<script>
import * as d3 from "d3";

export default {
    props: [
        'title', 
        'hierarchy', 
        'width', 
        'translation', 
        'labelOptions',
    ],

    data(){
        return {
            currentDepth:0,
            selection: null,
            hovered: {
                name: ' ',
            },
            graph: null,
        }
    },

    mounted(){
        // `arc()` factory function is being assigned as a method of the component
        this.arc = d3.arc()
            .startAngle(d => d.x0)
            .endAngle(d => d.x1)
            .padAngle(d => Math.min((d.x1 - d.x0)/2, 0.005))
            .padRadius(this.radius * 1.5)
            .innerRadius(d => d.y0 * this.radius)
            .outerRadius(d => Math.max(d.y0 * this.radius, 
                            d.y1 * this.radius - 1))

        // svg translation 
        const translation = `translate(
            ${this.width*this.translation.horizontal}, 
            ${this.width*this.translation.vertical}
        )`
        this.graph = d3.select(this.$el)
            .select('.graph')
            .append('svg')
            .attr('width', this.width * 7) 
            .attr('height', this.width * 6.5)
            .style('padding', '10px')
            .style('margin', '0px')
            .style('font', '12px sans-serif')
            .style('box-sizing', 'border-box')
            .append('g')
            .attr('transform', translation)
    },

    watch:{
        ready: {
            //immediate: true,
            handler(v){
                this.currentDepth = 0
                if(v){
                    // prepare the hierarchy into a D3 hierarchy
                    this.root = this.partition()
                    this.defineColorScale()
                    // clear the old graph
                    this.graph.selectAll('g').remove()
                    // build new graph
                    this.makeGraph()

                    // announce graph is complete, return what you want
                    this.$emit('ready', {
                        total: this.totalValue
                    })
                }
            }
        }
    },

    computed:{
        ready(){
            return this.graph && this.hierarchy
        },
        radius(){
            return this.width
        },
        totalValue(){
            return this.root.value
        },
    },

    methods: {
        defineColorScale(){
            const dataPoints = this.root.children.map(c=>c.data.id)
            this.colorScale = d3.scaleOrdinal()
                .domain(dataPoints)
                .range(
                    d3.quantize(
                        d3.interpolateViridis, //d3.interpolateRainbow, 
                        dataPoints.length
                    ).reverse()
                )
        },

        partitionData(p){
            return {
                name:p.data.name,
                id:p.data.id,
                depth:p.depth,
                value: p.value,
                x0:p.x0,
                x1:p.x1,
                y0:p.y0,
                y1:p.y1,
            }
        },

        coords({x0=0, x1=0, y0=0, y1=0}){
            return {x0,x1,y0,y1}
        },

        makeGraph(){
            // root node is the focus of a new graph   
            this.setSelection(this.root)

            this.root.each(d=>{
                /* `current` is used for positioning, we're interested in
                   coordinate information.
                   Also see later definition of `target` to understand how both
                   are used.
                */
                //if(d.depth<=this.selection.depth+2){
                //    // we want the current coords here
                //    // (x0, x1, y0, y1). also see `target` 
                //    d.current = this.coords(d)
                //} else {
                //    // default coords at origin
                //    d.current = this.coords({})
                //}
                d.current = this.coords(d)
                d.label = this.truncateOnWord(
                    d.data.name, 
                    this.labelOptions.characterLimit
                )
            })

            this.graph.selectAll('g').remove()

            this.initPaths()
            //this.initTitles()
            this.initCenter()
            this.initBackButton()
            //this.initLabels()
        },

        setSelection(node){
            this.selection = this.partitionData(node)
            this.selection.children = node.children.map(this.partitionData)
            this.$emit('selected', this.selection)
        },

        initPaths(){
            const v = this
            // root of graph is a circle 
            v.opacity = d=>d.children ? .45 : .25
            v.paths = this.graph.append('g')
                .attr('fill-opacity', 0.45)
                .selectAll('path')
                .data(v.root.descendants().slice(1))
                .join('path')
                .attr('fill', d=> {
                    while(d.depth > 1){
                        // the top parent determines the color
                        d = d.parent
                    }
                    return this.colorScale(d.data.id)
                })
                .attr('fill-opacity', d=>{
                    d.arcVisible = v.arcVisible(d.current) 
                    return d.arcVisible ? v.opacity(d) : 0
                })
                .attr('d', d=> d.current && v.arc(d.current))
                .on('mouseover', d=>{
                    if(d.arcVisible){
                        v.setLabel(d)
                        //v.$set(this.hovered, 'name', d.data.name)
                        d3.select(this.$el).selectAll(`text.label${d.data.id}`)
                            .text(d.data.name)
                    }
                })
                .on('mouseout', d=>{
                    v.setLabel()
                    //v.$set(this.hovered, 'name', ' ')
                    d3.select(this.$el).selectAll(`text.label${d.data.id}`).text(d.label)
                })

            // only arcs with children are clickable
            v.paths.filter(d => d.children)
                .style('cursor', 'pointer')
                .on('click', v.selected)
        },

        initTitles(){
            const v = this
            const format = d3.format(',d')
            // title should appear on hover
            v.paths.append('title')
                .text(d => {
                    const ancestors =  d.ancestors().map(d => d.data.name)
                        .reverse()
                    ancestors.shift()
                    const path = ancestors.join('\n')
                    return `${path}\n${format(d.data.value)}`
                })
        },

        setLabel(d){
            const v = this
            let label = ' '
            if (d){
                const percent = Math.round(d.value/v.totalValue * 10000)/100
                label = d.data.name + ' ' + percent + '%' 
            } 
            v.$set(v.hovered, 'name', label)
        },

        initCenter(){
            const v = this
            v.center = this.graph.append('g').append('circle')
                .datum(v.root)
                .attr('r', v.radius * .75)
                .attr('fill', '#eee')
                .attr('fill-opacity', .6)
                .on('mouseover', d=>{
                    v.setLabel(d)
                })
                .on('mouseout', d=>{
                    v.setLabel()
                    //v.$set(v.hovered, 'name', ' ')
                })
                //.attr('pointer-events', 'all')
                //.on('click', v.selected)
        },

        updateCenter(d, t){
            const v = this
            const c = this.center
            let color = null
            let opacity = null
            // the parent category determines the ultimate color,
            // so we bubble up to the parent (i.e. depth==1).
            let p = d
            while (p.depth > 1){
                p = p.parent
            }
            // if depth is 0 (i.e. root category) we also use that as color
            // else we fetch the color from the scale
            color = p.depth && this.colorScale(p.data.id) || '#eee'
            // root element (color==null) is almost transparent, 
            // whereas other elements are at .6 
            opacity = (p.depth && .6) || .8

            // interpolators for color and opacity
            const icolor = d3.interpolate(c.attr('fill'), color)
            const iopacity = d3.interpolate(c.attr('fill-opacity'), opacity)

            // center transition
            c.datum(d)
                .transition(t)
                .attrTween('fill-opacity', ()=>t=>(color && iopacity(t)) || opacity)
                .attrTween('fill', ()=>t=>icolor(t))
        },

        labelTransform(d){
            if(!d){
                return
            }
            // rotation angle is determined by calculating mid-point between
            // the arc's start and end angles, then converting it to degrees
            let x = (d.x0 + d.x1) / 2 * 180 / Math.PI
            // 0 deg starts at 12 O'Clock in svg
            // whereas transform's places 0 deg at 3 O'Clock
            // so we rotate the system back 90 deg to express in
            // transform what we've expressed previously in svg. 
            x -= 90

            // label's translation (i.e. linear motion):
            // move text away from center to place it midpoint between 
            // the inner and outer radius of its corresponding arc
            const y = (d.y0 + d.y1) / 1.4 * this.radius

            // a final rotation is added to re-orient upside down texts (180 deg).
            // it needs to be placed after the translation, which is why 
            // it's not included in the calculation above.
            return `
                rotate(${x}) 
                translate(${y}, 0) 
                rotate(${x < 90 ? 0 : 180})
                `
        },

        labelVisible(d, coords){
            // if labels are explicitly disabled: no labels
            if(this.labelOptions.off){
                return false
            }
            // if coordinates of data point to the center: no labels
            if(!(coords.x0 || coords.x1)){
                return false
            }
            // if only label up to a specific level visible in the chart must show
            if(this.labelOptions.depth){ 
                if((d.depth-this.labelOptions.depth)>this.currentDepth){
                    return false
                }
            }
            const c = coords
            const bigEnough = (c.x1 - c.x0) * (c.y1 - c.y0) > 0.2
            return c.y1 <= 3 && c.y0 >= 1 && bigEnough
        },

        truncateOnWord(str, limit) {
            if(str===undefined) return

            const trimmable = '\u0009\u000A\u000B\u000C\u000D\u0020\u00A0' + 
                '\u1680\u180E\u2000\u2001\u2002\u2003\u2004\u2005\u2006' + 
                '\u2007\u2008\u2009\u200A\u202F\u205F\u2028\u2029\u3000\uFEFF'
            const reg = new RegExp('(?=[' + trimmable + '])')
            const words = str.split(reg)
            let count = 0
            const rv = words.filter(function(word) {
                count += word.length
                return count <= limit
            }).join('')

            if (rv===str){
                return rv
            }
            return rv + '...'
        },

        partition(){ 

            // see details of calculations here:
            // https://github.com/d3/d3-hierarchy#node_sum
            // https://github.com/d3/d3-hierarchy#node_count

            /* calculate value of nodes, then sort each node's children. 
               
               If a node is a leaf in the original hierarchy, its value is 
               simply returned, else (if it contains children), its value is
               calculated as the sum of its children's value.
            */

            const root = d3.hierarchy(this.hierarchy)
                //.sum(d => d.leaf?d.value:null)
                .sum(d => d.value)
                .sort((a,b) => b.value - a.value)

            /* Conceptually, the sunburst is a mapping of a hierachy to a pie.

               The pie is made of concentric circles, with each circle 
               representing a level in the hierarchy. Each circle in turn is
               subdivided into arc portions, where each portion represents a
               node at that specific hierarchy level. The length of each arc is
               relative to the value associated with the node.
                
               The respective maxima of the sunburst pie are 2PI radians (i.e
               a full circle) and its number of circles (i.e. the number of 
               levels in the hierarchy). Thus we partition the node in the
               hierarchy with respect to those maxima.
            */
            d3.partition()
                .size([2 * Math.PI, root.height])(root)
            return root
        },

        arcVisible(d){
            if (!d){
                return false
            }
            return true
            //return d.y1 <= 3 && d.y0 >= 1 && d.x1 > d.x0
        },

        initBackButton(){
            const v = this
            
            const circle = this.graph.append('g')
                .append('circle')
                .attr('r', v.radius *.4)
                .attr('fill', '#eee')
                .attr('fill-opacity', .8)

            this.backButton = this.graph.append('g')
                .append('svg:image')
                .style('display', 'none')
                .style('cursor', 'pointer')
                .attr('xlink:href', '/back.svg')
                .attr('x', -this.width * .20)
                .attr('y', -this.width * .20)
                .attr('width', this.width * .40)
                .attr('height', this.width * .40)
                .attr('pointer-events', 'all')
                .on('click', v.selected)
        },

        updateBackButton(current){
            this.backButton.datum(current || this.root)
            if(!current){
                this.backButton
                    .style('display','none')
            } else {
                this.backButton
                    .style('display','block')
            }
        },

        selected(p){
            const v = this
            // datum of center of graph is either 
            // - the parent of this data
            // - or the root data, if no parent
            //parent.datum(p.parent || v.root)
            this.updateBackButton(p.parent)
            this.currentDepth = p.depth
            this.setSelection(p || v.root)
            this.$emit('selected', this.selection)


            // calculate target coordinates for each data item
            this.root.each(d => {
                if(v.isDescendant(d, p)){
                    d.target = v.newTarget(d, p)
                } else {
                    // set target to origin
                    d.target = this.coords({})
                }
            })

            p.target = {x0:0, x1:Math.PI*2, y0:.5, y1:1}

            const t = v.graph.transition().duration(750)

            this.paths
                .transition(t)
                .tween('data', d =>{
                    const i = d3.interpolate(d.current, d.target)
                    // this is the function that updates the 
                    // current position over time.
                    return t => d.current = i(t)
                })
                .filter(d=> d.arcVisible || v.arcVisible(d.target))
                .attr('fill-opacity', d => {
                    d.arcVisible = v.arcVisible(d.target)
                    return d.arcVisible ? v.opacity(d) : 0
                })
                .attrTween('d', d => () => {
                    return v.arc(d.current)
                })

            this.updateCenter(p, t)

            this.labels.filter(function(d){
                // filter labels whose visibility needs to be updated:
                // - either they're currently visible
                // - or their target should be visible
                return d.labelVisible || v.labelVisible(d, d.target)
            }).transition(t)
                .attr('fill-opacity', d => {
                    d.labelVisible = v.labelVisible(d, d.target)
                    return +d.labelVisible
                })
                // track the current position as it's being updated over time 
                // (see earlier function that gradually updates d.current)
                .attrTween('transform', d=> t=>this.labelTransform(d.current))
        },

        isDescendant(d, p){

            // this function compares equalities while accounting 
            // for floating point inconsistencies in JavaScript

            // d.x0 < p.x0 
            // => d.x0 - p.x0 < 0  i.e. d.x0 - p.x0 must be lt 0
            // to account for possible floating point errors
            // replace 0 with a really close negative number: -1e-14
            if(d.x0 - p.x0 < -1e-14){
                return false
            }

            // d.x1 > p.x1
            // => d.x1 - p.x1 > 0  i.e. d.x1 - p.x1 must be gt 0
            // replace 0 with a really close positive number: 1e-14
            if(d.x1 - p.x1 > 1e-14){
                return false
            }
            if (d.depth <= p.depth){
                return false
            }

            return true
        },

        newTarget(d, p){
            const pi2 = 2 * Math.PI
            const ratio = 1/(p.x1 - p.x0) 
            return {
                x0: (d.x0-p.x0) * ratio * pi2,
                x1: (d.x1-p.x0) * ratio * pi2,
                y0: d.y0 - p.depth,
                y1: d.y1 - p.depth, 
            }
        },

        initLabels(){
            this.labels = this.graph.append('g')
                .attr('pointer-events', 'none')
                .attr('text-anchor', 'middle')
                .style('user-select', 'none')
                .selectAll('text')
                .data(this.root.descendants().slice(1).filter(d=>d.current))
                .join('text')
                .attr('dy', '0.5em')
                .attr('class', d=>`label${d.data.id}`)
                .text(d=>d.label)
                .attr('fill-opacity', d=> {
                    d.labelVisible = this.labelVisible(d, d.current)
                    return +d.labelVisible
                })
                .attr('transform', d=> this.labelTransform(d.current))
        }

    },

}
</script>

<style>
.abso{
    position: absolute;
    top: 500px;
    left: 10px;
}
</style>
