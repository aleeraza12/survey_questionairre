<template>
  <!-- <div id="app"> -->
    <!-- <img alt="Vue logo" src="./assets/logo.png"> -->
    <SurveyForm_v2 msg="Welcome to Your Vue.js App"/>
  <!-- </div> -->
</template>

<script>
import SurveyForm_v2 from './components/SurveyForm.vue'

export default {
  name: 'App',
  components: {
    SurveyForm_v2
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
