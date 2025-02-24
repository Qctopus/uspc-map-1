<template>
  <div class="map-and-disclaimer-container">
    <div class="map-container">
      <l-map :zoom="2" :center="[51.505, -0.09]" style="height: 400px" ref="mapRef">
        <l-tile-layer :url="arcgisTileUrl"></l-tile-layer>
        <!-- Add other map elements here as needed -->
      </l-map>
    </div>
    <div class="map-disclaimer">
      The designations employed and the presentation of material on this map do not imply the expression of any opinion whatsoever on the part of the Secretariat of the United Nations or UNDP concerning the legal status of any country, territory, city or area or its authorities, or concerning the delimitation of its frontiers or boundaries.
      References to Kosovo* shall be understood to be in the context of UN Security Council resolution 1244 (1999).
    </div>
  </div>
</template> 

<script>
import { ref, watch, onMounted, defineComponent } from 'vue';
import { LMap, LTileLayer } from 'vue3-leaflet';
import L from 'leaflet';

export default defineComponent({
  components: {
    LMap,
    LTileLayer,
  },
  props: {
    resetSelection: Boolean,
    projects: Array,
  },
  setup(props, { emit }) {
    const mapRef = ref(null);
    const arcgisTileUrl = 'https://geoservices.un.org/arcgis/rest/services/ClearMap_WebTopo/MapServer/tile/{z}/{y}/{x}?blankTile=false';
    let geoLayer = null;
    const selectedCountries = new Set();
    let countryFunding = {};

    const getColor = (funding) => {
  const maxFunding = Math.max(...Object.values(countryFunding));
  const ratio = funding / maxFunding;
  
    // Lightness varies from 50% (light grey) to 20% (dark grey)
    // Adjust these values if you want a different range of greys
    const lightness = 60 - (5 * ratio);

    return `hsl(0, 0%, ${lightness}%)`;
  };

    const updateCountryStyle = (layer, countryCode) => {
      if (selectedCountries.has(countryCode)) {
        layer.setStyle({
          fillColor: 'green', // Selected color
          weight: 1,
          opacity: 1,
          fillOpacity: 0.6,
        });
      } else {
        layer.setStyle({
          fillColor: getColor(countryFunding[countryCode] || 0),
          weight: 0,
          opacity: 0,
          fillOpacity: countryFunding[countryCode] ? 0.6 : 0,
        });
      }
    };

    const selectCountry = (countryCode) => {
      if (geoLayer) {
        geoLayer.eachLayer((layer) => {
          const code = layer.feature.properties.ISO_A2;
          if (code === countryCode) {
            if (selectedCountries.has(countryCode)) {
              selectedCountries.delete(countryCode);
            } else {
              selectedCountries.add(countryCode);
            }
            updateCountryStyle(layer, countryCode);
          }
        });
      }
      emit('select-country', Array.from(selectedCountries));
    };

    const clearSelection = () => {
      if (geoLayer) {
        geoLayer.eachLayer((layer) => {
          const countryCode = layer.feature.properties.ISO_A2;
          selectedCountries.delete(countryCode);
          updateCountryStyle(layer, countryCode);
        });
      }
      selectedCountries.clear();
    };

    onMounted(async () => {
      if (!mapRef.value) {
        console.error("Map container is not yet available");
        return;
      }
      const leafletMap = mapRef.value.$data.map;

      const projectResponse = await fetch('/SDG_2023.json');
      const projectData = await projectResponse.json();

      countryFunding = {};
      projectData.projects.forEach(project => {
        const budget = parseFloat(project.budget.replace(/[^0-9.]/g, ''));
        const countryCode = project.country_code;
        countryFunding[countryCode] = (countryFunding[countryCode] || 0) + budget;
      });

      const geoResponse = await fetch('/countries.json');
      const geoData = await geoResponse.json();

      geoLayer = L.geoJSON(geoData, {
        style: feature => {
          const countryCode = feature.properties.ISO_A2;
          const funding = countryFunding[countryCode] || 0;
          return {
            fillColor: getColor(funding),
            weight: 0,
            opacity: 0,
            fillOpacity: funding > 0 ? 0.6 : 0,
          };
        },
        onEachFeature: (feature, layer) => {
          if (countryFunding[feature.properties.ISO_A2]) {
            layer.on('click', () => {
              const countryCode = feature.properties.ISO_A2;
              selectCountry(countryCode);
            });
          }
        },
      }).addTo(leafletMap);
    });

    watch(() => props.resetSelection, (newValue) => {
      if (newValue) {
        clearSelection();
        emit('selection-reset');
      }
    });

    return { mapRef, arcgisTileUrl };
  },
});
</script>

<style scoped>
.map-container {
  height: 600px;
  position: relative;
}

.map-disclaimer {
  padding-top: 10px;
  padding-bottom: 20px; /* Add some space below the disclaimer */
  text-align: left;
  font-size: 0.8em;
}
</style>
