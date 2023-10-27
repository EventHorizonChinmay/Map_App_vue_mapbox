<script>
import search from './assets/search.gif'
import Map from "./components/Map.vue";
import "../node_modules/mapbox-gl/dist/mapbox-gl.css"
let height = window.innerHeight
export default {
  components: {
    Map,
  },

  data: () => ({
    location: {
      lng: 77.641622,
      lat: 12.911872,
      bearing: 0,
      pitch: 0,
      zoom: 17.5,
    },
    search,
    height: window.innerHeight,
    currentStyleIndex: 1, // Track the index of the current map style
    mapStyles: [
      "mapbox://styles/mapbox/satellite-streets-v11",
      "mapbox://styles/mapbox/streets-v12",
    ],
  }),

  methods: {
    // Add the toggleMapStyle method
    toggleMapStyle() {
      this.currentStyleIndex =
        (this.currentStyleIndex + 1) % this.mapStyles.length;
    },
  },
};

</script>

<template>
  <div id="layout">

    <div id="sidebar">
      <!-- Windows Height:  <p>{{height}}</p> -->
      Longitude: <input v-model="lng" type="number"/>
      Latitude: <input v-model="lat"  type="number"/> |
      <div class="btncontainer">
      <button @click="location = { lng:lng  , lat:lat, zoom: 7, pitch:0, bearing: 0};
      lng=''; lat=''" id="searchbtn">🔍
      <!-- <img src="search"/> -->
    </button> </div>
      <!-- 🔎 -->
      <div class="btncontainer">
      <button @click="location = {lng: 77.641622, lat: 12.911872, zoom: 16.5, pitch:0 , bearing: 0 }" id="resetbtn">🔁</button>
    </div>
      <!-- ↻ -->
    </div>
    <div id="sidebar2">
      Longitude: <b style="font-weight: 600;margin:10px">{{location.lng.toFixed(2) }}</b> |
      Latitude: <b style="font-weight: 600; ;margin:10px">{{ location.lat.toFixed(2) }}</b> |
      Zoom: <b style="font-weight: 600;margin-left:10px; ">{{ location.zoom.toFixed(1) }} X</b> 
    </div>
    <div id="errorMsg">
      {{ Math.abs(lng)>180 || Math.abs(lat)>90 ? "Outside the limits of Earth" :'' }}
    </div>
    <Map
      v-model="location"
      :style="mapStyles[currentStyleIndex]"
    />
  </div>
</template>

<style>
#layout {
  flex: 1;
  display: flex;
}

#sidebar2 {
  background-color: rgba(255, 255, 255, 0.5);
  border: 5px solid rgba(255,255,255,0.1);
  color: rgba(0, 0, 0, 0.9);
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
}
#sidebar input{
  margin-left: 5px;
  margin-right: 5px;
}
#sidebar {
  /* background-color: rgba(35, 55, 75, 0.9); */
  background-color: rgba(255, 255, 255, 0.8);
  color: rgba(0, 0, 0, 0.9);
  height: 50px;
  width: 360px;
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
  background-color: black;

}
#searchbtn, .searchbtn, .resetbtn, #resetbtn{
  background-color: rgba(35 ,55, 76,0);
  /* background-color: rgba(255,255,255,9); */
  border: 1px solid rgba(255,255,255,0.5);
  /* box-shadow: .5px 0.5px 1px rgba(200,200,200,0.8); */
  padding: 0px;
  border-radius: 5px;
  cursor: pointer;
  margin: 5px;
}
#searchbtn:hover, .searchbtn:hover, .resetbtn:hover, #resetbtn:hover{
  border: 2px solid rgba(0,0,0,0.1);
  box-shadow: 1px 1px 1px rgba(0,0,0,0.1);
  /* background-color: black; */
}
</style>