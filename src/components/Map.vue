<template>
  <div v-if="mapLoadingError" class="error-message">
      <p>Sorry, something seems to be broken. Try again later...</p>
    </div>
  <div v-else ref="mapContainer" class="map-container">
    

      <button id="toggle_button" title="Toogle Map Style" @click="toggleStyle"> <span class="location">
        <!-- Toggle Map Style -->
      </span> </button>
      <div >
        <button 
        id="drop_marker_button" alt="OK" title="OK" v-if="markerMode" 
        @click="toggleMarkerMode">
        <!-- {{ markerMode ? '✔️' :  "📌" }}  -->
        ✔️
        </button> 
      </div>
      <button 
      id="drop_marker_button" alt="Drop Pins" v-if="!markerMode" class="pinBtns" title="Drop Pins"
      @click="toggleMarkerMode">
      <!-- {{ markerMode ? '✔️' :  "📌" }}  -->
      <!-- 📌 -->
      </button>

      <button
        v-if="markers.length > 0 && !markerMode"
        id="drop_marker_button"
        @click="clearFeatures" title="Remove Pins"
      >
      ❌
      </button>


      <!-- <div ref="searchdiv" id="search-div"></div> -->
      
      <!-- <div id="drawingTools">  -->
        <button
          v-if="!drawingMode && markers.length <= 0" title="Draw on the map"
          id="start_drawing_button"
          class="drawModeStart drawingTools"
          @click="startDrawing"
        > 
        <!-- 🖋️ -->
        </button>
        <button
          v-if="!drawingMode && markers.length > 0" title="Clear drawings "
          id="clear_features_button drawingTools"  class="drawingTools"
          @click="clearFeatures"
        >
        <!-- 🗑️ -->
        ❌
        </button>

        <button
          v-if="drawingMode"
          id="end_drawing_button"
          class="drawingTools" title="OK"
          @click="endDrawing"
        >
        <!-- 👍🏼 -->
        ✔️
        </button>
      <!-- </div> -->
      <div ref="searchdiv" id="search-div"></div>
      
<div v-if="showInfoBox&& !markerMode &&  (markers.length === 2 || polygonLayerId)" class="info-box">
  <p v-if="markers.length === 2 " style="font-weight: 600;">Distance: <span style="font-weight: 800; font-size:small;"> {{ distance<1 ? (distance*1000).toFixed(3)+ ' meters' : distance.toFixed(3) +' kms' }} </span></p>
  <p v-if="polygonLayerId" style="font-weight: 600;">Area &nbsp  &nbsp &nbsp:<span style="font-weight: 800; font-size:small;"> &nbsp {{ area < 1e6 ? area.toFixed(3) + ' meters' : (area/1e6).toFixed(3) + ' kms' }}<sup style="font-weight:800">2</sup> </span> </p>
  <p v-if="polygonLayerId" style="font-weight: 600;">Perimeter : &nbsp <span style="font-weight: 800;  font-size:small;">{{ perimeter<1 ? perimeter.toFixed(3)*1000 + ' meters' : perimeter.toFixed(3) +' kms' }} </span></p>
</div>
      <button class="saveGeom" v-if="markers.length>0" @click="saveGeometry" title="Save Geometry">
        <!-- Save Geometry -->
      </button>
    
      <div class="savedGeomList" v-show="savedGeometries.length > 0" v-if="savedGeometries.length > 0" >
        <div class="GeomList">
          <h3 style="margin-bottom: 12px; text-align: center; font-weight: 700; font-size: medium;">Saved Geometries</h3>
          <div  v-for="(geometry, index) in savedGeometries" :key="index" 
          class="GeomEntry hidden">
            
            <button id="deleteGeomName" @click="deleteSavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion !== geometry.name">🗑️ </button>
            
            <button id="deleteGeomName" @click="deleteSavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion === geometry.name" class="hidden">🗑️ |</button>
            
            <button id="GeomName" @click="displaySavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion !== geometry.name">{{ geometry.name }}</button>
            
            <button id="GeomName" @click="displaySavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion === geometry.name" class="hidden">{{ geometry.name }}</button>
          </div>

          <div  v-for="(geometry, index) in savedGeometries" :key="index" class="">
            
            <button id="deleteGeomName" @click="deleteSavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion === geometry.name" class="hidden">🗑️ |</button>
            
            <button id="GeomName" @click="displaySavedGeometry(geometry.name)" v-if="selectedGeometryForDeletion === geometry.name" class="hidden">{{ geometry.name }}</button>
          </div>

      </div>


  </div>
  <!-- <div style="position: absolute; bottom: 0px ; right:20px; width: fit-content;"><a href="https://www.flaticon.com/free-animated-icons/location" title="location animated icons">Location animated icons created by Freepik - Flaticon</a> </div> -->
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
      styleNo: 1,
      mapStyles: [
        "mapbox://styles/mapbox/streets-v12",
        "mapbox://styles/mapbox/satellite-streets-v11",
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
      selectedGeometryForDeletion: null,
      hovered: false,
      mapLoadingError: false,
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
    map.on("error", (e) => {
      console.error(e);
      this.mapLoadingError = true;
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
        map.setCenter({ lng: next.lng.toFixed(3), lat: next.lat.toFixed(3) });
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
      popupContent.innerHTML = `<p>Lat: ${coordinates.lat.toFixed(3)}</p><p> Lng: ${coordinates.lng.toFixed(3)}</p>`;

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
      this.distance = distance;
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
      this.distance = distance;
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

      marker.getElement().classList.add('custom-marker'); 

      marker.index = markerIndex; 
      this.markers.push(marker);


      if (this.markers.length === 2) {
        console.log('-------------------------',this.markers.length,'-------------------------')
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
        this.distance = distance
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
      this.removePolyline()

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
        this.clearFeatures();
        this.removePolyline()

        selectedGeometry.markers.forEach(coords => {
          const popupContent = document.createElement('div');
          popupContent.innerHTML = `<p>Lat: ${coords.lat}</p><p>Lng: ${coords.lng}</p>`;

          const marker = new mapboxgl.Marker()
            .setLngLat(coords)
            .setPopup(new mapboxgl.Popup().setDOMContent(popupContent))
            .addTo(this.map);

          this.markers.push(marker);
        });
        console.log('-0-0-0-0-0-0-0-0-0-0-0-0-0-0',selectedGeometry.lineCoordinates)
        if (selectedGeometry.polygonCoordinates) {
          this.drawPolygonFeature(selectedGeometry.polygonCoordinates);
        } else if (selectedGeometry.lineCoordinates) {
          this.drawPolyline(selectedGeometry.lineCoordinates);
          
        }

        if (selectedGeometry.polygonCoordinates) {
          this.showInfoBox = true;
          this.area = this.calculatePolygonArea(selectedGeometry.polygonCoordinates);
          this.perimeter = this.calculatePolygonPerimeter(selectedGeometry.polygonCoordinates);
        } else if (selectedGeometry.lineCoordinates) {
          this.showInfoBox = true;
          const startPoint = selectedGeometry.markers[0];
          const endPoint = selectedGeometry.markers[1];
          console.log(startPoint, endPoint)
          // const distance = turf.distance(startPoint, endPoint) * 1000;
          // this.distance = distance;

          // ///////////////////////////////////////
          ///////////////////////////////////////////
            //   const startPoint = this.markers[0].getLngLat();
            // const endPoint = this.markers[1].getLngLat();

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
            this.distance = distance
              ///////////////////////////////////////////
          console.log(distance)
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
        this.selectedGeometryForDeletion = name;
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
    font-family: 'Chivo Mono', sans-serif;
    flex: 1;
  }

  /* #toggle_button {
    background-image: url('https://cdn.supsystic.com/wp-content/uploads/2018/08/Different-Map-Styles-3-1024x444.png');
    background-image: url('@/assets/toggleMapStyle2.png');
    background-size: cover;
    font-family: 'Chivo Mono', sans-serif;
    z-index: 1;
    position: absolute;
    bottom: 0px;
    right: 0;
    height: 60px;
    margin: 30px;
    cursor: pointer;
    transition: 0.2s ease;
    border-radius: 50%;
    text-shadow: 0 0 5px white;
    box-shadow: -5px 2px 5px black;
    color: black;
    font-weight: 500;
    font-size: large;
    border: 3px solid black;
  }
  #toggle_button:hover{

    transition: 0.2s ease;
    background-image: url('@/assets/toggleMapStyle.png');
    background-size: cover;
    clip-path: polygon(0 0, 100% 0, 100% 100%, 0 100%);
    cursor: pointer;
    border-radius: 50%;
    box-shadow: 5px -2px 5px black;
    text-shadow: 0 0 5px white;
    color: black;
    font-weight: 500;transition: 0.2s ease;

  } */
  #toggle_button{
    position: absolute;
    bottom: 0px;
    right: 0;
    margin-right:auto;
    /* margin: none; */
    /* padding: none; */
    width: 60px;
    height: 60px;
    box-shadow: none;
    background-size: contain;
    background-image: url('@/assets/toggleMapStyle.png');
    z-index: 1;
    position: absolute;
    bottom: 0px;
    right: 0;
    margin: 30px;
    cursor: pointer;
    transition: 0.2s ease;
    border-radius: 50%;
    text-shadow: 0 0 5px white;
    box-shadow: -5px 2px 5px black;
    color: black;
    font-weight: 500;
    font-size: large;
    border: 3px solid black;
  }
  #toggle_button:hover{
    background-size: contain;
    background-image: url('@/assets/toggleMapStyle2.png'); 
    border-radius: 50%;
    box-shadow: 5px -2px 5px black;
  }

  #searchdiv {
    height: fit-content;
    font-family: 'Chivo Mono', sans-serif;
    width: fit-content;
    /* margin-right: 50px; */
    background-color: white;
    border: 1px solid rgba(100,100,100,1);
    box-shadow:0 0 5px rgba(100,100,100,1);
  }

  .mapboxgl-ctrl-geocoder {
    padding-top: 15px;
    padding-bottom: 15px;
    padding-left: 15px ;
    z-index: 1;
    margin-right: 50px;
    /* position: absolute;
    top: 0px;
    left: 800px; */
    border-radius: 5px;
    background-color:rgba(0255,0255,255,0.9);
    /* background-color: red; */
    width: 260px;
    border: 1px solid rgba(100,100,100,1);
    box-shadow: 0 0 5px rgba(100,100,100,1);

  }
  .mapboxgl-ctrl-geocoder--input{
    padding:15px;
    padding-right: 0;
  }
  .mapboxgl-ctrl-geocoder{
    position: absolute;
    border-radius: 10px;
    right: 10px;
    /* background-color: red; */
  }
  .mapboxgl-ctrl-geocoder--button{
    position: absolute;
    top:-35px;
    right: -10px;
    width: 35px;
    height: 35px;
    /* background-color: green; */

    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    cursor: pointer;
  }
  .mapboxgl-ctrl-geocoder--button:hover{
    background: white;
    background-size: cover;
    border: 2px solid red;
  }
  input{
    font-family: 'Chivo Mono', sans-serif;
    border-radius: 5px;
  }
  .mapboxgl-ctrl-geocoder--suggestion{
    
    cursor: pointer;
    color:rgb(30, 29, 29);
    padding: 2px;
    border: solid 1px rgba(0,0,0,0.1);
    border-radius: 5px;
  }
  .mapboxgl-ctrl-geocoder input{
    width: max-content;

  }
  .mapboxgl-ctrl-geocoder .mapboxgl-ctrl-icon {
    /* position: absolute; */
    /* background-color: red; */
    top: 50%;
    /* right: 100px;  */
    border-radius: 10px;
    transform: translateY(-50%);
  }
  a{
    color: white;
  }
  .suggestions, .mapboxgl-ctrl-geocoder--suggestion li{
    color:black;
    margin-left: -20px;
    margin-right: 5px;
    margin-top:5px;
    /* box-shadow: 0 1px 5px rgba(0,0,0,0.5); */
    /* width: 20px; */
  }
  li{
    /* border:5px solid red */
    margin-left:5px;
  }
  .mapboxgl-ctrl-geocoder--suggestion-title{
    font-weight: 900;
  }
  
  .mapboxgl-ctrl-geocoder--icon-search {
    margin-right: 10px;
    margin-top: -10px;
    padding-top: 15px;
    /* padding-bottom: -5px; */
    height: 30px;
    width: auto;
    /* width: 50px; */
    /* background-color: white; */
    /* position: absolute; */
    /* top: 5px; */
    /* margin : 5px; */

    /* z-index: 1; */
  }
  .mapboxgl-ctrl-geocoder--icon-loading{
  display: none;
}
  #drop_marker_button {
    z-index: 1;
    font-family: 'Chivo Mono', sans-serif;
    background-color: rgba(0205,0205,0205,01.8);
    /* background-color: red; */
    position: absolute;
    top: 80px;
    left: 0px;
    height: 35px;
    margin-left: 30px;
    cursor: pointer;
    /* border: 1px solid rgba(100,100,100,1); */
    transition: 0.2s ease ;
    width: 40px;
    height: 40px;

    background-color: white;
    background-size: cover;
    box-shadow: 0px 0px 10px white;
    /* border: none; */
    /* border: 2px white solid; */
    border-radius: 5px;
    cursor: pointer;
  }

  .pinBtns{
    width: 40px;
    height: 40px;
    background-image: url('@/assets/pin.png'); 
    background-color: white;
    background-size: cover;
    border: 2px solid rgba(0,0,0,0.5);
    
  }
  .pinBtns:hover{
    
    border: 1px solid rgba(0,0,0,0.5);
  box-shadow: 1px 1px 2px rgba(0,0,0,0.5);
  padding: 2px;
    width: 40px;
    height: 40px;
    border: 2px solid rgb(100,100,100);
    background-image: url('@/assets/pin.gif'); 
    /* font-family: 'Chivo Mono', sans-serif; */
    animation: playGif 1s infinite;
    transition: 0.2s ease;
}


  .drawingTools{
    width: 40px;
    height: 40px;
    background-size: cover;
    z-index: 1;
    font-family: 'Chivo Mono', sans-serif;
    cursor: pointer;
    transition: 0.2s ease ;
    position: absolute;
    top: 80px;
    left: 50px;
    margin-left: 30px;
    /* border: 5px solid white; */
    transition: 0.2s ease ;
    z-index: 1;
    background-color: white;
    background-size: cover;
    box-shadow: 0px 0px 10px white;
    cursor: pointer;
    border-radius: 5px;
    cursor: pointer;
    border: 1px solid rgba(100,100,100,1);
    transition: 0.2s ease ;
    z-index: 1;
  }

  
  .info-box {
  background-color: rgba(255, 255, 255, 0.90);
  color: rgba(0, 0, 0, 0.9);
  font-family: 'Chivo Mono', sans-serif;
  /* font-weight: 900; */
  width: 280px;
  align-items: center; /* Center align vertically */
  padding: 6px 12px;
  margin: 12px;
  z-index: 3;
  position: absolute;
  bottom: 70px;
  left: 20px;
  border: 1px solid rgba(100,100,100,1);
  padding: 10px;
  margin-top: 10px;
  box-shadow: 0 0 10px white;
  border-radius: 10px;
  }

.custom-marker {
  border-radius: 50%;
  width: 20px;
  height: 20px;
  /* border: 1px solid black; */
  cursor: pointer;
}

.saveGeom{
  z-index: 1;
  font-family: 'Chivo Mono', sans-serif;
  position: absolute;
  top: 50px;
  left : 100px;
  /* top: 40px; */
}
 .showSavedGeom{
  z-index: 1;
  font-family: 'Chivo Mono', sans-serif;
  position: absolute;
  top: 200px;
}

.savedGeomList {
  z-index: 1;
  font-family: 'Chivo Mono', sans-serif;
  position: absolute;
  top: 140px;
  left: 20px;
  max-height: 500px;
  overflow-y: auto;
  padding: 10px;
  /* margin: 50px; */
  height: 50%;
  /* max-height: 80%; */
  border-radius: 10px;
  width: 300px;
  border: 1px solid rgba(100,100,100,1);
  /* width:fit-content; */
  padding-bottom: 50px;
  background-color: white;
  box-shadow: 0 0 10px rgba(100,100,100,1);
}

.hidden {
  display: none;
}


.GeomList{
  justify-content: space-between;
  border: none;
  padding: 10px;
  height: fit-content;
  font-family: 'Chivo Mono', sans-serif;
  background-color: rgba(255,255,255,10);
  max-height: 20rem;
  width: fit-content;
  margin: auto;
  /* padding-bottom: 5rem; */
  /* margin-left: 20px; */
}

.GeomEntry{
  display: flex;
  margin: auto;
  font-family: 'Chivo Mono', sans-serif;
  margin-bottom: 15px;
  /* height: 32px; */
  min-height: 0;
  /* padding-bottom: 5px; */
  justify-content: space-between;
  background-color: rgba(0,0,0,0);
  border-bottom: 1px rgba(0, 0, 0, 0.2) solid;
  border-right: 1px rgba(0, 0, 0, 0.2) solid;
  box-shadow: 1px 1px 5px rgba(0,0,0,0.2);
}
.GeomEntry:hover{
  /* box-shadow: 0 0 3px rgba(0,0,0,0.8); */
  min-height: 10%;
  background-color: rgba(0,0,0,0.2);
}

#deleteGeomName{
  border: none;
  background-color: rgba(0,0,0,0);
  width: 30px;
  height: 20px;
  cursor: pointer;
  margin: auto;
  color: rgba(0,0,0,5.8)
}

#deleteGeomName:hover{
  font-size: larger;
  text-shadow: 0 0 1px rgba(0,0,0,0.5);

  transition: 0.1s ease-in-out;
  /* background-color: red; */
}

#GeomName{
  text-align: left;
  border: none;
  /* border-bottom: 1px solid gray; */
  width: 180px;
  font-family: 'Chivo Mono', sans-serif;
  max-width: 100%;
  /* background-color: rgba(255,255,255,10); */
  color: blue ;
  cursor: pointer;
  /* white-space: nowrap; */
  overflow: hidden;
  text-overflow: ellipsis;
  
}

#GeomName:hover{
  font-size: larger;
  text-shadow: 0 0 1px rgba(0,0,0,0.2);
  transition: 0.1s ease-in-out;
}

.GeomName::after {
  content: '\00a0-\00a0'; /* Adds a non-breaking space and a dash before the word */
}


#start_drawing_button{
    width: 40px;
    height: 40px;
    background-size: cover;
    z-index: 1;
    font-family: 'Chivo Mono', sans-serif;
    background-color: rgba(0205,0205,0205,01.8);

    cursor: pointer;
    transition: 0.2s ease ;
    border-radius: 5px;
    cursor: pointer;
    border: 1px solid black;
    transition: 0.2s ease ;
    z-index: 1;
    background-color: white;
    background-size: cover;
    box-shadow: 0px 0px 5px white;
    cursor: pointer;
    /* border:none; */

    background-image: url('@/assets/blueprint.png'); 
  }
  #start_drawing_button:hover{
    background-color: rgba(255, 255 , 255, 1);
    border: 2px solid rgb(100,100,100);
    background-image: url('@/assets/blueprint.gif'); 
    font-family: 'Chivo Mono', sans-serif;
    transition: 0.2s ease ;
  }

  .search-div #search-div{
    border: 1px solid rgba(100,100,170,1);
    background-color: red;
  }
  .saveGeom{
  background-color: white;
  background-size: contain;
    background-image: url('@/assets/save-file (1).png'); 
    height:40px;
    width: 40px;
    margin-left: 30px;
    border-radius: 5px;
    box-shadow: 0 0 10px rgba(255,255,255,2);
    background-color: rgba(0205,0205,0205,01.8);
    margin: 30px;
    cursor: pointer;
    transition: 0.2s ease ;
    border-radius: 5px;
    cursor: pointer;
    border: 1px solid black;
    transition: 0.2s ease ;
    z-index: 1;
    background-color: white;
    background-size: cover;
    box-shadow: 0px 0px 5px white;

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
.error-message {
  color: red;
  font-size: 16px;
  font-weight: bold;
  text-align: center;
  margin-top: 20px;
}



@media screen and (max-width: 800px) {
.mapboxgl-ctrl-geocoder {
    position: absolute;
    top: 00px;
    /* padding-top: 15px; */
    padding-bottom: 15px;
    padding-left: 15px ;
    z-index: 1;
    margin-right: 50px;
    border-radius: 10px;
    background-color:rgba(0255,0255,255,0.9);
    /* background-color: red; */
    width: 260px;
    border: 1px solid rgba(100,100,100,1);
    box-shadow: 0 0 5px rgba(100,100,100,1);
  }

  .savedGeomList{
    margin-top: -10px;
    max-height: 50%;
  }
  .GeomList{
    width: 100%;
  }
  #toggle_button{
    position: absolute;
    bottom: 0rem;
    left:1rem;
    margin-right:auto;
    /* margin: none; */
    /* padding: none; */
    width: 60px;
    height: 60px;
    box-shadow: none;
    background-size: contain;
    background-image: url('@/assets/toggleMapStyle.png');
  }
  #toggle_button:hover{
    background-size: contain;
    background-image: url('@/assets/toggleMapStyle2.png'); 
    box-shadow: none;
  }
  .info-box{
    position: absolute;
    bottom: 5rem;
    right: 10px;
    width: 260px;
    margin-left: auto;
  }
  .mapboxgl-marker .mapboxgl-marker-anchor-center{
    cursor:crosshair;}
}
</style>
