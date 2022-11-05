<template>
  <GanttChart></GanttChart>
</template>

<script>
import GanttChart from "@/components/GanttChart";

export default {
  name: 'App',
  components: {
    GanttChart
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
