<template>
  <div ref="mapContainer" class="map-container">
    <button id="toggle_button" @click="toggleStyle"><img src=""/> Toggle Map Style</button>
    <p></p>
    <div ref="searchdiv" id="search-div"></div>
  </div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import MapboxGeocoder from "@mapbox/mapbox-gl-geocoder";

mapboxgl.accessToken = `pk.eyJ1IjoiY2hpbm1heWciLCJhIjoiY2xvNWc3NnF3MGF6ZzJxcWRtdTdmbGVsdyJ9.PyfJ7zptOqULwNQ5eB7RUQ`;

export default {
  props: ["modelValue"],
  data() {
    return {
      map: null,
      styleNo: 0,
      mapStyles: [
        "mapbox://styles/mapbox/satellite-streets-v11",
        "mapbox://styles/mapbox/streets-v12",
        "mapbox://styles/mapbox/outdoors-v11",
      ],
    };
  },
  mounted() {
    const { lng, lat, zoom, bearing, pitch } = this.modelValue;

    const map = new mapboxgl.Map({
      container: this.$refs.mapContainer,
      style: this.mapStyles[this.styleNo],
      center: [lng, lat],
      bearing,
      pitch,
      zoom: zoom,
    });

    const updateLocation = () => this.$emit("update:modelValue", this.getLocation());

    map.on("move", updateLocation);
    map.on("zoom", updateLocation);
    map.on("rotate", updateLocation);
    map.on("pitch", updateLocation);

    // Add the Geocoder control
    const geocoder = new MapboxGeocoder({
      container: this.$refs.searchdiv,
      accessToken: mapboxgl.accessToken,
      mapboxgl: mapboxgl,
    });
    map.addControl(geocoder);

    this.map = map;
  },
  watch: {
    modelValue(next) {
      const curr = this.getLocation();
      const map = this.map;

      if (curr.lng != next.lng || curr.lat != next.lat)
        map.setCenter({ lng: next.lng, lat: next.lat });
      if (curr.pitch != next.pitch) map.setPitch(next.pitch);
      if (curr.bearing != next.bearing) map.setBearing(next.bearing);
      if (curr.zoom != next.zoom) map.setZoom(next.zoom);
    },
  },
  methods: {
    getLocation() {
      return {
        ...this.map.getCenter(),
        bearing: this.map.getBearing(),
        pitch: this.map.getPitch(),
        zoom: this.map.getZoom(),
      };
    },
    toggleStyle() {
      this.styleNo = (this.styleNo + 1) % this.mapStyles.length;
      this.map.setStyle(this.mapStyles[this.styleNo]);
    },
  },
};
</script>

<style>
  .map-container {
    flex: 1;
  }

  #toggle_button {
    z-index: 1;
    position: absolute;
    bottom: 0px;
    right: 0;
    height: 35px;
    margin: 30px;
    /* border-radius: 5px; */
    cursor: pointer;
  }

  #searchdiv {
    height: fit-content;
    width: fit-content;
    background-color: white;
  }

  .mapboxgl-ctrl-geocoder {
    padding: 15px;
    z-index: 1;
    /* position: absolute;
    top: 0px;
    left: 800px; */
    border-radius: 5px;
    background-color:rgba(0255,0255,255,0.8);
    width: 250px;
    /* min-width: 250px; */
    
    /* max-width: 350px; */
    /* display: flex; */
  }
input{
  border-radius: 5px;
}
.mapboxgl-ctrl-geocoder--suggestion{
  cursor: pointer;
  padding: 2px;
  border: solid 1px rgba(0,0,0,0.1);
  border-radius: 5px;
}
  .mapboxgl-ctrl-geocoder input{
    width: max-content;

  }
  .mapboxgl-ctrl-geocoder .mapboxgl-ctrl-icon {
    /* position: absolute; */
    background-color: red;
    /* top: 50%; */
    /* right: 100px;  */
    /* transform: translateY(-50%); */
  }
</style>
