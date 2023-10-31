
<template>
  <div id="layout">

    <div id="sidebar">

      <span class="location">Latitude:</span> <input v-model="lat"  type="number" title="Enter lattitude" placeholder="Lat"/> 
      <span class="location">Longitude:</span> <input v-model="lng" type="number" title="Enter longitude" placeholder="Lng"/>&nbsp|&nbsp
      <div class="btncontainer">
      <button @click="location = { lng:lng  , lat:lat, zoom: 7, pitch:0, bearing: 0};
      lng=''; lat=''" id="searchbtn" title="Search the coordinate"> 
    </button> </div>
      <!-- 🔎 -->
      <div class="btncontainer">
      <button title="Refresh Map" @click="location = {lng: 77.641622, lat: 12.911872, zoom: 16.5, pitch:0 , bearing: 0 }" id="resetbtn"></button>
    </div>
      <!-- ↻ -->
    </div>

    <div id="sidebar2" title="Current location">
      <span class="location"> Latitude:</span> <b style="font-weight: 600; ;margin:10px">{{ location.lat.toFixed(2)+',' }}</b>
      <span class="location">Longitude:</span>  <b style="font-weight: 600;margin:10px">{{location.lng.toFixed(2) }}</b> |
      <span class="location">Zoom:</span> <b style="font-weight: 600;margin-left:10px; ">{{ location.zoom.toFixed(1) }} X</b> 
    </div>

    <div id="errorMsg">
      {{ Math.abs(lng)>180 || Math.abs(lat)>90 ? "Outside the limits of Earth" :'' }}
    </div>
    
    <!-- @transitionComplete="hideIntroScreen"  -->
    <IntroScreen v-if="showIntroScreen"  
    @transitionComplete="hideIntroScreen"
    
    />
    <Map  v-model="location"
      :style="mapStyles[currentStyleIndex]"/>

  </div>
</template>

<script>
import search from './assets/search.gif'
import Map from "./components/Map.vue";
import "../node_modules/mapbox-gl/dist/mapbox-gl.css"
import IntroScreen from './components/IntroScreen.vue'
// import Map from '@/components/Map.vue'

let height = window.innerHeight
export default {
  components: {
    IntroScreen,
    Map,
  },
  mounted() {
    setTimeout(() => {
      this.showIntroScreen = false;
    }, 6000);
  },

  data: () => ({
    location: {
      lng: 77.641622,
      lat: 12.911872,
      bearing: 0,
      pitch: 0,
      zoom: 17.5,
      isFirstLoad: true,
    },
    search,
    showIntroScreen: true,
    height: window.innerHeight,
    currentStyleIndex: 1, // Track the index of the current map style
    mapStyles: [
      "mapbox://styles/mapbox/satellite-streets-v11",
      "mapbox://styles/mapbox/streets-v12",
    ],
    
  }),

  methods: {
    toggleMapStyle() {
      this.currentStyleIndex =
        (this.currentStyleIndex + 1) % this.mapStyles.length;
    },
    hideIntroScreen() {
      this.showIntroScreen = false;
    }
  },
};

</script>


<style>
#layout {
  font-family: 'Chivo Mono', sans-serif;
  flex: 1;
  display: flex;
}

#sidebar2 {
  background-color: rgba(255, 255, 255, 0.9);
  border: 1px solid rgba(0, 0, 0, 0.5);
  box-shadow: 0px 0px 10px white;
  color: rgba(0, 0, 0, 0.9);
  font-family: 'Chivo Mono', sans-serif;

  font-weight: 500;
  padding: 6px 18px;
  z-index: 1;
  height: 50px;
  position: absolute;
  bottom: 0px;
  left: 0px;
  margin: 12px;
  border-radius: 10px;
  display: flex; /* Use flexbox */
  align-items: center; /* Center align vertically */

  box-shadow: 0 0 10px rgba(0,0,0,0.5);
}
#sidebar input{
  margin-left: 5px;
  margin-right: 5px;
}
#sidebar {
  background-color: rgba(255, 255, 255, 0.9);
  border: 1px solid rgba(0, 0, 0, 0.5);
  box-shadow: 0px 0px 10px rgba(100,100,100,1);
  color: rgba(0, 0, 0, 0.9);
  font-family: 'Chivo Mono', sans-serif;
  height: 50px;
  width:440px;
  /* color: #fff; */
  display: flex; 
  align-items: center; /* Center align vertically */
  padding: 6px 12px;
  /* font-family: monospace; */
  z-index: 1;
  position: absolute;
  top: 0;
  left: 0;
  margin: 12px;
  border-radius: 10px;
  font-family: 'Chivo Mono', sans-serif;

}
.btncontainer{
  width: 50px;
}
#errorMsg {
  color: red;
  padding: 6px 12px;
  /* font-family: monospace; */
  z-index: 1;
  position: absolute;
  top: 0px;
  left: 700px;
  margin: 12px;
  border-radius: 4px;
}
input{
  width: 60px;
  height: 30px;
}

#sidebar .sidebar #btncontainer .btncontainer{
  width: 50px;
  border: 1px solid rgba(100,100,100,1);
  box-shadow: 2px 2px 2px rgba(100,100,100,1);
  background-color: black;
}

input::placeholder{
  width: 100%;
  text-align: center;  
  margin-left: 10px;
}

#resetbtn, .resetbtn{
  width: 30px;
  height: 30px;
  background-image: url('@/assets/refresh.png'); 
  background-color: white;
  background-size: cover;
  /* border: none; */
  /* border: 2px white solid; */
  border-radius: 5px;
  cursor: pointer;
}
#resetbtn:hover{
  width: 35px;
  height:35px;
  background-image: url('@/assets/refresh.gif'); /* Replace with your GIF path */
  animation: playGif 1s infinite;
  transition: 0.2s ease;
}

 /* .resetbtn, #resetbtn */
 #searchbtn, .searchbtn {
  width: 30px;
  height: 30px;
  background-image: url('@/assets/search.png'); 
  background-color: white;
  background-size: contain;
  /* border: none; */
  /* border: 2px white solid; */
  border-radius: 5px;
  cursor: pointer;
}
#searchbtn:hover{
  width: 35px;
  height:35px;
  background-image: url('@/assets/search.gif'); /* Replace with your GIF path */
  animation: playGif 1s infinite;
  transition: 0.2s ease;
}
#searchbtn:hover, .searchbtn:hover, .resetbtn:hover, #resetbtn:hover{
  border: 1px solid rgba(100,100,100,1);
  box-shadow: 1px 1px 2px rgba(0,0,0,0.5);
  padding: 2px;
  /* background-color: black; */
}

#drop_marker_button,.drop_marker_button, #start_drawing_button, #clear_features_button , .drawingTools, .saveGeom{
    box-shadow: 0 0 1px black;
    border: 1px solid rgba(0,0,0,0.5);
    /* background-color: red; */
  }

#drop_marker_button:hover,.drop_marker_button:hover, #start_drawing_button:hover, #clear_features_button:hover , .drawingTools:hover , .saveGeom:hover{
    box-shadow: 0 0 4px rgba(0,0,0,0.5);
  }



@media screen and (max-width: 800px) {
#sidebar, .sidebar , .sidebar2, #sidebar2{
  display: flex;
    position: absolute;
    /* top: 10px; */
    right: 10px;
    width: 260px;
    /* background-color: red; */
    margin-right: 10px;
    margin-left: auto;
    border: 1px solid rgba(0,0,0,0.5);
    box-shadow: 0 0 5px rgba(0,0,0,0.5);

  }

  .location{
    display: none;
  }
  #sidebar{
    /* display: flex; */
      position: absolute;
      top: 5rem;
  }
  #clear_features_button , .drawingTools{
    position: absolute;
    top: 10rem;
    right:7rem;
    border: 1px solid black;
    box-shadow: 0 0 10px white;
    border-radius: 5px;
    /* background-color: red; */
    margin-left:auto;
  }
  #drop_marker_button,.drop_marker_button, #start_drawing_button, #clear_features_button , .drawingTools, .saveGeom{
    box-shadow: 0 0 1px rgba(0,0,0,0.5);
    border: 1px solid rgba(0,0,0,0.5);
    /* background-color: red; */
  }
  #drop_marker_button:hover,.drop_marker_button:hover, #start_drawing_button:hover, #clear_features_button:hover , .drawingTools:hover, .saveGeom:hover{
    box-shadow: 0 0 4px black;
  }

  #drop_marker_button,.drop_marker_button, #start_drawing_button  {
    position: absolute;
    top: 10rem;
    right:10rem;
    margin-left:auto;
  }
  #start_drawing_button, #end_drawing_button{
    position: absolute;
    top: 10rem;
    right:7rem;
    margin-left:auto;
  }
  .saveGeom{
    position: absolute;
    top: 130px;
    right:2rem;
    margin-left:auto;
    margin: none;
    padding: none;
    /* background-color: red; */
  }
  .savedGeomList{
    position: absolute;
    top: 14rem;
    right:2rem;
    margin-left:auto;
    margin: none;
    padding: none;
    width: 200px;
  }
  .GeomList{
    /* background-color: red; */
    text-align: center;
  }
  .GeomEntry{
    margin: auto;
    margin-bottom: 5px;
    width:80%;
  }
  /* #toggle_button{
    position: absolute;
    bottom: 0rem;
    left:2rem;
    margin-right:auto;
    margin: none;
    padding: none;
    width: 70px;
    color: rgba(0,0,0,0);
    text-shadow: rgba(0,0,0,0)
  }
  #toggle_button:hover{
    color: rgba(0,0,0,0);
    text-shadow: rgba(0,0,0,0)
  } */
}

@keyframes playGif {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.05);
  }
  100% {
    transform: scale(1);
  }
  }
</style>