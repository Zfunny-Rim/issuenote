<template>
  <HelloWorld/>
</template>
<script>
import HelloWorld from './components/IssueNote.vue';
//import Header from './views/layout/HeaderView.vue';
//import Footer from './views/layout/FooterView.vue';

export default {
  name: 'App',
  components: {
    HelloWorld,
   // Header,
    //Footer,
    //Topbar
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
