<template>
    <div ref="mapContainer" class="map-container">
      <button id="toggle_button" @click="toggleStyle">Toggle View</button>
    </div>
  </template>
  
  <script>
  import mapboxgl from "mapbox-gl";
  mapboxgl.accessToken = `pk.eyJ1IjoiY2hpbm1heWciLCJhIjoiY2xvNWc3NnF3MGF6ZzJxcWRtdTdmbGVsdyJ9.PyfJ7zptOqULwNQ5eB7RUQ`;
  
  export default {
    props: ["modelValue"],
    data() {
      return {
        map: null,
        styleNo: 0,
        mapStyles: [
          "mapbox://styles/mapbox/satellite-v9",
          "mapbox://styles/mapbox/streets-v12",
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
    top: 0px;
    left: 90%;
    margin: 12px;
  }
  </style>
  