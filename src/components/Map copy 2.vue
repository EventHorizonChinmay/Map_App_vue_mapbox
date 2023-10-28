<template>
  <div ref="mapContainer" class="map-container">
    <button id="toggle_button" @click="toggleStyle">Toggle Map Style</button>
    <!-- <button v-if="markerMode || markers.length <= 0"
     id="drop_marker_button" 
     @click="toggleMarkerMode">{{ markerMode ? '✔️' :  "📌" }} 
    </button>
    <button
      v-if="markers.length > 0 && !markerMode"
      id="drop_marker_button"
      @click="clearMarkers"
    >
    ❌
    </button> -->
    
    <div ref="searchdiv" id="search-div"></div>
    
    <div id="drawingTools"> 
      <button
        v-if="!drawingMode && markers.length <= 0"
        id="start_drawing_button"
        @click="startDrawing"
      >
        Start Drawing
      </button>
      <button
        v-if="!drawingMode && markers.length > 0"
        id="clear_features_button"
        @click="clearFeatures"
      >
        Clear Features
      </button>
      <button
        v-if="drawingMode"
        id="end_drawing_button"
        @click="endDrawing"
      >
        End Drawing
      </button>
     </div>
    <div ref="searchdiv" id="search-div"></div>
    
    <div v-if="showInfoBox && (markers.length === 2 || polygonLayerId)" class="info-box">
      <p v-if="markers.length === 2">Distance: {{ distance }} kilometers</p>
      <p v-if="polygonLayerId">Area: {{ area }} square meters</p>
      <p v-if="polygonLayerId">Perimeter: {{ perimeter }} kilometers</p>
    </div>
  </div>
</template>

<script>
import mapboxgl from "mapbox-gl";
import MapboxGeocoder from "@mapbox/mapbox-gl-geocoder";
import * as turf from '@turf/turf';

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
      markerMode: false,
      markers: [],
      distancesArray: [], 
      markerCoords: [], 
      lineLayerId: null,
      drawingMode: false,
      polygonLayerId: null,
      area: null,
      perimeter: null,
      showInfoBox: false,
      distance:null,
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

    toggleMarkerMode() {
      this.markerMode = !this.markerMode;
      console.log('toggleMarjerMode')
      if (this.markerMode) {
        this.map.on("click", this.dropMarker);
      } else {
        this.map.off("click", this.dropMarker);
        if (this.markers.length == 2) {
          this.drawSegment();
          const coordinates = this.markers.map(marker => marker.getLngLat());

          const R = 6371; // Radius of the Earth in kilometers
          const lat1 = coordinates[0].lat * Math.PI / 180;
          const lat2 = coordinates[1].lat * Math.PI / 180;
          const lon1 = coordinates[0].lng * Math.PI / 180;
          const lon2 = coordinates[1].lng * Math.PI / 180;

          const dLat = lat2 - lat1;
          const dLon = lon2 - lon1;

          const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
                    Math.cos(lat1) * Math.cos(lat2) *
                    Math.sin(dLon/2) * Math.sin(dLon/2);
          const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));

          const distance = R * c;
          this.showInfoBox = true;
          this.distance = distance.toFixed(6);

          console.log(`Distance between points: ${distance} kilometers`);

          const lineFeature = {
            type: 'Feature',
            geometry: {
              type: 'LineString',
              coordinates: [
                [coordinates[0].lng, coordinates[0].lat],
                [coordinates[1].lng, coordinates[1].lat]
              ]
            }
          };

          if (this.lineLayerId) {
            this.map.removeLayer(this.lineLayerId);
            this.map.removeSource('line');
          }

          this.lineLayerId = 'line-' + Date.now(); // Generate a unique ID for the line layer
          this.map.addSource('line', {
            type: 'geojson',
            data: lineFeature
          });
        }
      }
    },
    dropMarker(e) {
      const coordinates = e.lngLat;
      const popupContent = document.createElement('div');
      popupContent.innerHTML = `<p>Lat: ${coordinates.lat}</p><p> Lng: ${coordinates.lng}</p>`;

      const deleteButton = document.createElement('button');
      deleteButton.textContent = 'Delete';

      const markerIndex = this.markers.length;

      deleteButton.addEventListener('click', () => this.deleteMarker(markerIndex)); 
      popupContent.appendChild(deleteButton);

      const marker = new mapboxgl.Marker()
        .setLngLat(coordinates)
        .setPopup(new mapboxgl.Popup().setDOMContent(popupContent))
        .addTo(this.map);

      marker.index = markerIndex; // Store the index as a property of the marker

      this.markers.push(marker);
      if (this.markers.length === 2) {
        this.drawSegment();
      }
    },

    deleteMarker(index) {
      const marker = this.markers.find(marker => marker.index === index);
      if (this.markers.length < 2) {
        this.removeLine();
      }
      if (marker) {
        marker.remove();
        this.markers = this.markers.filter(marker => marker.index !== index);
      }
      
    },
    drawSegment() {
      const coordinates = this.markers.map(marker => marker.getLngLat());

      const lineFeature = {
        type: 'Feature',
        geometry: {
          type: 'LineString',
          coordinates: [
            [coordinates[0].lng, coordinates[0].lat],
            [coordinates[1].lng, coordinates[1].lat]
          ]
        }
      };

      this.lineLayerId = 'line-' + Date.now();

      this.map.addSource('line', {
        type: 'geojson',
        data: lineFeature
      });

      this.map.addLayer({
        id: this.lineLayerId,
        type: 'line',
        source: 'line',
        layout: {
          'line-join': 'round',
          'line-cap': 'round'
        },
        paint: {
          'line-color': '#888',
          'line-width': 10
        }
      });
      console.log('Line Added')
    },

    removeLine() {
      if (this.lineLayerId) {
        this.map.removeLayer(this.lineLayerId);
        this.map.removeSource('line');
        this.lineLayerId = null; // Reset lineLayerId
      }
    },
    clearMarkers() {
      this.markers.forEach((marker) => marker.remove());
      this.markers = [];
      this.markerCoords = [];
      this.removeLine();
      this.distancesArray = [];
    },
    startDrawing() {
      console.log('Drawing Segment ');

      if (this.markers.length === 2) {
        const coordinates = this.markers.map(marker => marker.getLngLat());

        const R = 6371; // Radius of the Earth in kilometers
        const lat1 = coordinates[0].lat * (Math.PI / 180);
        const lat2 = coordinates[1].lat * (Math.PI / 180);
        const lon1 = coordinates[0].lng * (Math.PI / 180);
        const lon2 = coordinates[1].lng * (Math.PI / 180);

        const dLat = lat2 - lat1;
        const dLon = lon2 - lon1;

        const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
          Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
        const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

        const distance = R * c;
        this.distance = distance.toFixed(2); // Set the distance value
        this.showInfoBox = true; // Show the info-box

        console.log(`Distance between points: ${distance} kilometers`);
      } else {
        this.distance = null; // Reset distance if there are not exactly 2 markers
        this.showInfoBox = false; // Hide the info-box
      }

      this.drawingMode = true;
      this.map.on("click", this.drawPolygon);
    },

    endDrawing() {
      this.drawingMode = false;
      this.map.off("click", this.drawPolygon);
      this.showInfoBox = true;
    },

    drawPolygon(e) {
      const coordinates = e.lngLat;
      const popupContent = document.createElement('div');
      popupContent.innerHTML = `<p>Lat: ${coordinates.lat}</p><p> Lng: ${coordinates.lng}</p>`;

      const deleteButton = document.createElement('button');
      deleteButton.textContent = 'Delete';

      const markerIndex = this.markers.length;

      deleteButton.addEventListener('click', () => this.deleteMarker(markerIndex)); 
      popupContent.appendChild(deleteButton);

      const marker = new mapboxgl.Marker()
        .setLngLat(coordinates)
        .setPopup(new mapboxgl.Popup().setDOMContent(popupContent))
        .addTo(this.map);

      marker.index = markerIndex; // Store the index as a property of the marker

      this.markers.push(marker);

      // Check if there are at least 3 markers to form a polygon
      if (this.markers.length >= 3) {
        this.drawPolygonFeature();
        this.showInfoBox = true; 

      }
    },

    drawPolygonFeature() {
  const coordinates = this.markers.map(marker => marker.getLngLat());
  if (coordinates.length==2){
    // ///////////////////////////////////////////////////////
    // toggleMarkerMode()
    // drawSegment()
    ///////////////////////////////////////////////////////
    console.log('two')
  }
  if (coordinates.length >= 3) {
    coordinates.push(coordinates[0]); // Close the polygon

    const polygonFeature = {
      type: "Feature",
      geometry: {
        type: "Polygon",
        coordinates: [coordinates.map(coord => [coord.lng, coord.lat])]
      }
    };

    if (this.polygonLayerId) {
      this.map.removeLayer(this.polygonLayerId);
      this.map.removeSource("polygon");
    }

    this.polygonLayerId = "polygon-" + Date.now();

    this.map.addSource("polygon", {
      type: "geojson",
      data: polygonFeature
    });

    this.map.addLayer({
      id: this.polygonLayerId,
      type: "fill",
      source: "polygon",
      layout: {},
      paint: {
        "fill-color": "#088",
        "fill-opacity": 0.4
      }
    });

    // Calculate area
    const area = this.calculatePolygonArea(coordinates);
    console.log(`Area of the polygon: ${area} square meters`);

    // Calculate perimeter
    const perimeter = this.calculatePolygonPerimeter(coordinates);
    console.log(`Perimeter of the polygon: ${perimeter} kilometers`);
  } else {
    console.error("At least 3 markers are required to form a polygon.");
  }
  this.area = this.calculatePolygonArea(coordinates);

// Calculate perimeter
this.perimeter = this.calculatePolygonPerimeter(coordinates);

},

calculatePolygonPerimeter(coordinates) {
  const lineString = turf.lineString(coordinates.map(coord => [coord.lng, coord.lat]));
  const length = turf.length(lineString, { units: "kilometers" });
  return length;
},



    calculatePolygonArea(coordinates) {
      const polygon = turf.polygon([coordinates.map(coord => [coord.lng, coord.lat])]); // Wrap coordinates in an array
      const area = turf.area(polygon);
      console.log(area);
      return area;
    },

    clearFeatures() {
      this.markers.forEach((marker) => marker.remove());
      this.markers = [];
      this.removePolygon();
    },

    removePolygon() {
      if (this.polygonLayerId) {
        this.map.removeLayer(this.polygonLayerId);
        this.map.removeSource('polygon');
        this.polygonLayerId = null; // Reset polygonLayerId
      }
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

  #drop_marker_button {
    z-index: 1;
    background-color: rgba(0205,0205,0205,01.8);
    /* background-color: red; */
    position: absolute;
    top: 40px;
    left: 0px;
    height: 35px;
    border-radius: 5px;
    margin: 30px;
    cursor: pointer;
    border: 1px solid rgba(100,100,100,1);
    transition: 0.2s ease ;
  }

  #drop_marker_button:hover{
    background-color: rgba(255, 255 , 255, 1);
    border: 2px solid rgb(100,100,100);
    transition: 0.2s ease ;
  }
  #drawingTools{
    z-index: 1;
    position: absolute;
    bottom: 30px;
    right: 0px;
    margin: 50px;
  }
  /* #start_drawing_button, #clear_features_button, #end_drawing_button{

  } */
  .info-box {
  background-color: rgba(255, 255, 255, 0.8);
  /* background-color: #f9f9f9; */
  color: rgba(0, 0, 0, 0.9);
  /* height: 50px; */
  width: 310px;
  /* color: #fff; */
  /* display: flex;  */
  align-items: center; /* Center align vertically */
  padding: 6px 12px;
  margin: 12px;
  z-index: 2;
  position: absolute;
  top: 80px;
  left: 50px;
  border: 1px solid #ccc;
  padding: 10px;
  margin-top: 10px;
  border-radius: 10px;
}
</style>
