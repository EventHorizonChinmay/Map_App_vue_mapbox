<script>
import Map from "./components/Map.vue";
import "../node_modules/mapbox-gl/dist/mapbox-gl.css"

export default {
  components: {
    Map,
  },

  data: () => ({
    location: {
      lng: 73.8567,
      lat: 18.5204,
      bearing: 0,
      pitch: 0,
      zoom: 9,
    },
    currentStyleIndex: 1, // Track the index of the current map style
    mapStyles: [
      "mapbox://styles/mapbox/satellite-v9",
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

      Longitude: <input v-model="lng" type="number" placeholder={+location.lng.toFixed(4)} />
      Latitude: <input v-model="lat"  type="number"/> |
      <button @click="location = { lng:lng  , lat:lat, zoom: 7, pitch:0, bearing: 0}">Search</button> -.-
      <button @click="location = { lng: 73.8567  , lat: 18.5204, zoom: 9, pitch:0 , bearing: 0 }">Reset</button>
      <!-- <button @click="location = { lng:   , lat: location.lat, zoom: location.zoom, pitch, bearing, styleBool:!location.styleBool }">2</button>
    -->
      <!-- <button @click="toggleMapStyle">Toggle Map Style</button> -->

    </div>
    <div id="sidebar2">
      Longitude: <b>{{location.lng.toFixed(4) }}</b> |
      Latitude: {{ location.lat.toFixed(4) }} |
      Zoom: {{ location.zoom.toFixed(2) }} |
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
  background-color: #fff;
  color: rgba(0, 0, 0, 0.9);
  padding: 6px 12px;
  /* font-family: monospace; */
  z-index: 1;
  position: absolute;
  top: 50px;
  left: 0px;
  margin: 12px;
  border-radius: 4px;
}
#sidebar {
  background-color: rgba(35, 55, 75, 0.9);
  color: #fff;
  padding: 6px 12px;
  /* font-family: monospace; */
  z-index: 1;
    
  position: absolute;
  top: 0;
  left: 0;
  margin: 12px;
  border-radius: 4px;
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
</style>