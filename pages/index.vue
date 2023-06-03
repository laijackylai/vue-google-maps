<template>
  <GoogleMap/>
</template>

<script lang="ts">
import Vue from 'vue'
import GoogleMap from '~/components/GoogleMap.vue';

export default Vue.extend({
    name: "IndexPage",
    components: { GoogleMap }
})
</script>
