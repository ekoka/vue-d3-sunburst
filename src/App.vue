<template>
<div id="app">
    <sunburst v-if="hierarchy" :hierarchy="hierarchy" />
</div>
</template>

<script>
import sunburst from '@/components/sunburst'

export default {
    name: 'app',

    data(){
        return {
            hierarchy: null,
        }
    },

    components: {
        sunburst
    },

    created(){
        this.$http.get('/data.json').then(response=>{
            this.hierarchy = response.body
            console.log(this.hierarchy)
        })
    },
}
</script>
