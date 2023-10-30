<template>
  <div ref="mapContainer" class="map-container">
    <div style="background-color: red;">
      
      <button id="toggle_button" @click="toggleStyle">Toggle Map Style</button>

      <button 
      id="drop_marker_button" alt="Drop pins"
      @click="toggleMarkerMode">
      {{ markerMode ? '✔️' :  "📌" }} 
      <!-- {{ markers.length>0 ? '✔️' :  "📌" }}  -->
      </button>

      <button
        v-if="markers.length > 0 && !markerMode"
        id="drop_marker_button"
        @click="clearFeatures"
      >
      ❌
      </button>


      <div ref="searchdiv" id="search-div"></div>
      
      <div id="drawingTools"> 
        <button
          v-if="!drawingMode && markers.length <= 0"
          id="start_drawing_button"
          @click="startDrawing"
        > 🖋️
          
        </button>
        <button
          v-if="!drawingMode && markers.length > 0"
          id="clear_features_button"
          @click="clearFeatures"
        >
        🗑️
        </button>
        <button
          v-if="drawingMode"
          id="end_drawing_button"
          @click="endDrawing"
        >
        👍🏼
        </button>
      </div>
      <div ref="searchdiv" id="search-div"></div>
      
      <div v-if="showInfoBox && (markers.length === 2 || polygonLayerId)" class="info-box">
        <p v-if="markers.length === 2">Distance: {{ distance<1 ? distance*1000+ ' meters' : distance +'kms' }} </p>
        <p v-if="polygonLayerId">Area: {{ area > 1e6 ? area + ' sq meters' : area/1e6 + ' sq kms' }} </p>
        <p v-if="polygonLayerId">Perimeter: {{ perimeter<1 ? perimeter*1000 + ' meters' : perimeter }} </p>
      </div>
      <button class="saveGeom" v-if="markers.length>0" @click="saveGeometry">Save Geometry</button>
    
      <button class="showSavedGeom" @click="showSavedGeometries">Show Saved Geometries</button>

      <div class="savedGeomList" v-if="savedGeometries.length > 0">
        <ul>
          <li v-for="(geometry, index) in savedGeometries" :key="index">
            <button @click="displaySavedGeometry(geometry.name)">{{ geometry.name }}</button>
            <button @click="deleteSavedGeometry(geometry.name)">Delete</button>
          </li>
        </ul>
      </div>
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
      showSavedGeometries: false, 
      lineLayerId: null,
      drawingMode: false,
      polygonLayerId: null,
      area: null,
      perimeter: null,
      showInfoBox: false,
      distance:null,
      savedGeometryNames: [],
      savedGeometries: [],
    };
  },
  computed: {
    savedGeometryNames() {
      const savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
      return savedGeometries.map(geometry => geometry.name);
    }
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
    this.savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
  
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

      marker.index = markerIndex;
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
      }}
      
    ,

    updateLocation() {
      const curr = this.getLocation();
      const map = this.map;

      if (curr.lng != map.getCenter().lng || curr.lat != map.getCenter().lat)
        map.setCenter({ lng: curr.lng, lat: curr.lat });
      if (curr.pitch != map.getPitch()) map.setPitch(curr.pitch);
      if (curr.bearing != map.getBearing()) map.setBearing(curr.bearing);
      if (curr.zoom != map.getZoom()) map.setZoom(curr.zoom);
    },

    getLocation() {
      return {
        ...this.map.getCenter(),
        bearing: this.map.getBearing(),
        pitch: this.map.getPitch(),
        zoom: this.map.getZoom(),
      };
    },

    startDrawing() {

      this.drawingMode = true;
      if (this.markers.length === 2) {
        this.drawLine()
        console.log('2')
        const coordinates = this.markers.map(marker => marker.getLngLat());
      const startPoint = coordinates[0];
      const endPoint = coordinates[1];

      const R = 6371; 
      const lat1 = startPoint.lat * (Math.PI / 180);
      const lat2 = endPoint.lat * (Math.PI / 180);
      const lon1 = startPoint.lng * (Math.PI / 180);
      const lon2 = endPoint.lng * (Math.PI / 180);

      const dLat = lat2 - lat1;
      const dLon = lon2 - lon1;

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
        Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      const distance = R * c;
      this.distance = distance.toFixed(2);
      this.showInfoBox = true;

      console.log(`Distance between points: ${distance} kilometers`);
      } else {
        this.distance = null;
        this.showInfoBox = false;
      }

      this.drawingMode = true;
      this.map.on("click", this.drawPolygon);
    },

    endDrawing() {
      this.drawingMode = false;
      this.map.off("click", this.drawPolygon);
      this.showInfoBox = true;
    },

    drawLine() {
      const coordinates = this.markers.map(marker => marker.getLngLat());
      const startPoint = coordinates[0];
      const endPoint = coordinates[1];

      const R = 6371;  
      const lat1 = startPoint.lat * (Math.PI / 180);
      const lat2 = endPoint.lat * (Math.PI / 180);
      const lon1 = startPoint.lng * (Math.PI / 180);
      const lon2 = endPoint.lng * (Math.PI / 180);

      const dLat = lat2 - lat1;
      const dLon = lon2 - lon1;

      const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
        Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

      const distance = R * c;
      this.distance = distance.toFixed(2);
      this.showInfoBox = true;

      console.log(`Distance between points: ${distance} kilometers`);
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

  marker.getElement().classList.add('custom-marker'); // Add custom class

  marker.index = markerIndex; 
  this.markers.push(marker);


  if (this.markers.length === 2) {
    const startPoint = this.markers[0].getLngLat();
    const endPoint = this.markers[1].getLngLat();

    const lineCoordinates = [startPoint, endPoint];
    this.drawPolyline(lineCoordinates);

    const R = 6371; 
    const lat1 = startPoint.lat * (Math.PI / 180);
    const lat2 = endPoint.lat * (Math.PI / 180);
    const lon1 = startPoint.lng * (Math.PI / 180);
    const lon2 = endPoint.lng * (Math.PI / 180);

    const dLat = lat2 - lat1;
    const dLon = lon2 - lon1;

    const a = Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) * Math.sin(dLon / 2);
    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    const distance = R * c;
    this.distance = distance.toFixed(2);
    this.showInfoBox = true;

    console.log(`Distance between points: ${distance} kilometers`);
  }

  if (this.markers.length >= 3) {
    this.removePolyline();
    this.drawPolygonFeature();
    this.showInfoBox = true;
  }
    },

    drawPolyline(coordinates) {
      const lineString = {
        type: "Feature",
        properties: {},
        geometry: {
          type: "LineString",
          coordinates: coordinates.map(coord => [coord.lng, coord.lat]),
        },
      };

      if (this.lineLayerId) {
        this.map.removeLayer(this.lineLayerId);
        this.map.removeSource("line");
      }

      this.lineLayerId = "line-" + Date.now();

      this.map.addSource("line", {
        type: "geojson",
        data: lineString,
      });

      this.map.addLayer({
        id: this.lineLayerId,
        type: "line",
        source: "line",
        layout: {},
        paint: {
          "line-color": "#ff8c00",
          "line-width": 2,
        },
      });
    },

    removePolyline() {
      if (this.lineLayerId) {
        this.map.removeLayer(this.lineLayerId);
        this.map.removeSource('line');
        this.lineLayerId = null;
      }
    },

    drawPolygonFeature() {
      const coordinates = this.markers.map(marker => marker.getLngLat());
      coordinates.push(coordinates[0]);

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

      this.area = this.calculatePolygonArea(coordinates);
      this.perimeter = this.calculatePolygonPerimeter(coordinates);
    },

    calculatePolygonArea(coordinates) {
      const polygon = turf.polygon([coordinates.map(coord => [coord.lng, coord.lat])]);
      const area = turf.area(polygon);
      return area;
    },

    calculatePolygonPerimeter(coordinates) {
      const lineString = turf.lineString(coordinates.map(coord => [coord.lng, coord.lat]));
      const length = turf.length(lineString, { units: "kilometers" });
      return length;
    },

    deleteMarker(index) {
      const marker = this.markers.find(marker => marker.index === index);
      if (this.markers.length < 2) {
        this.removePolygon();
      }
      if (marker) {
        marker.remove();
        this.markers = this.markers.filter(marker => marker.index !== index);
      }
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
        this.polygonLayerId = null;
      }
    },
    saveGeometry() {
      const savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
      
      const geometryName = window.prompt('Enter a unique name for the geometry:');
      
      if (geometryName !== null) {
        const isNameUnique = !savedGeometries.some(geometry => geometry.name === geometryName);
        
        if (isNameUnique) {
          const savedGeometry = {
            name: geometryName,
            markers: this.markers.map(marker => marker.getLngLat()),
            lineCoordinates: this.lineLayerId ? this.markers.map(marker => marker.getLngLat()) : null,
            polygonCoordinates: this.polygonLayerId ? this.markers.map(marker => marker.getLngLat()) : null,
          };
          savedGeometries.push(savedGeometry);
          localStorage.setItem('savedGeometries', JSON.stringify(savedGeometries));
          this.savedGeometries = savedGeometries;
          this.savedGeometries = savedGeometries;

        } else {
          alert('Geometry name must be unique. Please choose a different name.');
        }
      }
    },
    showSavedGeometries() {

      const savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
      
      if (savedGeometries.length === 0) {
        alert('No saved geometries found.');
        return;
      }
      
      const geometryNames = savedGeometries.map(geometry => geometry.name);
      // const selectedGeometryName = window.prompt('Select a geometry to display:\n\n' + geometryNames.join('\n'));
      
      // if (selectedGeometryName !== null) {
      //   const selectedGeometry = savedGeometries.find(geometry => geometry.name === selectedGeometryName);
      //   if (selectedGeometry) {
      //     this.displaySavedGeometry(selectedGeometry);
      //   } else {
      //     alert('Invalid geometry name. Please select a valid name.');
      //   }
      // }
    },
    displaySavedGeometry(name) {
      const savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
      const selectedGeometry = savedGeometries.find(geometry => geometry.name === name);

      if (selectedGeometry) {
        // Clear existing markers and geometries
        this.clearFeatures();

        // Recreate markers with popups
        selectedGeometry.markers.forEach(coords => {
          const popupContent = document.createElement('div');
          popupContent.innerHTML = `<p>Lat: ${coords.lat}</p><p>Lng: ${coords.lng}</p>`;

          const marker = new mapboxgl.Marker()
            .setLngLat(coords)
            .setPopup(new mapboxgl.Popup().setDOMContent(popupContent))
            .addTo(this.map);

          this.markers.push(marker);
        });

        // Draw the geometry based on saved coordinates
        if (selectedGeometry.polygonCoordinates) {
          this.drawPolygonFeature(selectedGeometry.polygonCoordinates);
        } else if (selectedGeometry.lineCoordinates) {
          this.drawPolyline(selectedGeometry.lineCoordinates);
        }

        // Calculate and display area/perimeter or distance
        if (selectedGeometry.polygonCoordinates) {
          this.showInfoBox = true;
          this.area = this.calculatePolygonArea(selectedGeometry.polygonCoordinates);
          this.perimeter = this.calculatePolygonPerimeter(selectedGeometry.polygonCoordinates);
        } else if (selectedGeometry.lineCoordinates) {
          this.showInfoBox = true;
          const startPoint = selectedGeometry.markers[0];
          const endPoint = selectedGeometry.markers[1];
          const distance = turf.distance(startPoint, endPoint) * 1000; // Convert to meters
          this.distance = distance.toFixed(2) + ' meters';
        }
      }
    },

    
    deleteSavedGeometry(name) {
      const confirmed = window.confirm(`Are you sure you want to delete "${name}"?`);
      if (confirmed) {
        const savedGeometries = JSON.parse(localStorage.getItem('savedGeometries')) || [];
        const updatedGeometries = savedGeometries.filter(geometry => geometry.name !== name);
        localStorage.setItem('savedGeometries', JSON.stringify(updatedGeometries));
        this.savedGeometries = updatedGeometries;
        this.savedGeometries = savedGeometries;
        this.$nextTick(() => {
          this.savedGeometryNames = updatedGeometries.map(geometry => geometry.name);
        });
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
    top: 50%;
    /* right: 100px;  */
    transform: translateY(-50%);
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
    background-color: rgba(0205,0205,0205,01.8);
    /* background-color: red; */
    position: absolute;
    top: 80px;
    left: 0px;
    height: 35px;
    border-radius: 5px;
    margin: 30px;
    width: 50px;
    cursor: pointer;
    border: 1px solid rgba(100,100,100,1);
    transition: 0.2s ease ;
    z-index: 1;
    /* position: absolute;
    top: 100px;
    left: 20px;
    margin: 12px; */
  }
  #drawingTools button{
    border: none;
    width: min-content;
    background-color: rgba(0, 0,0,0);
  }
  
  .info-box {
  background-color: rgba(255, 255, 255, 0.8);
  color: rgba(0, 0, 0, 0.9);
  width: 310px;
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
.custom-marker {
  /* Add your custom styles for the markers here */
  /* background-color: #ff0000; For example, change the marker color to red */
  border-radius: 50%;
  width: 20px;
  height: 20px;
  /* border: 1px solid black; */
  cursor: pointer;
}

.saveGeom{
  z-index: 1;
  position: absolute;
  top: 170px;
}
 .showSavedGeom{
  z-index: 1;
  position: absolute;
  top: 200px;
}
.savedGeomList {
  z-index: 1;
  position: absolute;
  top: 220px;
  background-color: rgba(255, 255, 255, 0.5);
  max-height: 500px;
  overflow-y: auto;
  padding: 10px;
  border-radius: 5px;
  width: 250px;
}
</style>
