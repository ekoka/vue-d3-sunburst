<template>
<div id="app">
    <sunburst v-if="hierarchy" 
        :hierarchy="hierarchy" 
        :translation="translation"
        :labelOptions="labelOptions"
        :width="width"
        />
</div>
</template>

<script>
import sunburst from '@/components/sunburst'
import * as d3 from "d3"

export default {
    name: 'app',

    data(){
        return {
            hierarchy: null,
            translation: {
                horizontal: 3, 
                vertical: 3, 
            },
            labelOptions: {
                depth:1,
                characterLimit:15,
            },
            width: 70,

        }
    },

    computed:{},

    components: {
        sunburst
    },

    created(){
        this.$http.get('/data.json').then(response=>{
            this.hierarchy = response.body
        })
    },
}
</script>
