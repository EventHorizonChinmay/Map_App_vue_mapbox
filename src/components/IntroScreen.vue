<template>
  <div class="intro-screen" 
  @click="transitionComplete" 
  style="cursor: pointer;">
    <div><h1 style="text-align: center; ">Chinmay's Map</h1>
    <p style="text-align: center; margin-top: 10px; font-weight: 550; ">For</p>
    <br/>
    <p style="text-align: center; "><img src="https://uploads-ssl.webflow.com/5d490ccd3cf49e0bd0ec972e/5d9f242549b5406b6852e112_skylark-drones-logo.png" alt="Skylark Drones" style=" width: auto;
  height: 100px; "/></p>
  <br/>
  <p style="text-align: center; font-size: small;"> &lt click anywhere to continue &gt </p>
</div>
    <!-- <button @click="transitionComplete" >Continue</button> -->
  
  </div>
</template>

<script>
export default {
  name: 'IntroScreen',
  methods: {
    transitionComplete() {
      this.$emit('transitionComplete');
    },
    dismissIntroScreen() {
      this.$emit('introScreenDismissed')
    }
    
  }
}
</script>

<style scoped>
.intro-screen {
    z-index: 3;
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  display: flex;
  margin: auto;
  justify-content: center;
  align-items: center;
  background-color: #ffffff;
  opacity: 1;
  transition: opacity 2s ease;
  background: radial-gradient(circle, white, black);
    

  /* position: relative;
  width: 300px;
  height: 200px;
  background: linear-gradient(to bottom, rgba(0, 0, 50, 0.8), rgba(0, 0, 50, 0.8)),
              linear-gradient(to top, rgba(0, 0, 50, 0.8), rgba(0, 0, 50, 0.8)),
              linear-gradient(to left, rgba(0, 0, 50, 0.8), rgba(0, 0, 50, 0.8)),
              linear-gradient(to right, rgba(0, 0, 50, 0.8), rgba(0, 0, 50, 0.8));
  border: 1px solid rgba(0, 0, 50, 0.8); */
}
.fade-out {
  opacity: 0.5;
  transition: opacity 2s; /* 2 seconds transition */
}
.container::after {
  content: '';
  z-index: 2;
  position: absolute;
  top: 10px;
  left: 10px;
  right: 10px;
  /* bottom:  */
}

h1 {
  font-family: 'Chivo Mono', sans-serif;
  font-size: 3rem; /* Adjust the font size as needed */
}
.intro-screen.fade-out {
  opacity:0.5;
  transition: opacity 2s ease;
  }
</style>