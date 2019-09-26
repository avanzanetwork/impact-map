<template>
  <div class="home">
    <div id="map"></div>
  </div>
</template>

<script>
// @ is an alias to /src
import axios from 'axios';
import eventData from '@/assets/eventData.json';

const mapboxgl = require('mapbox-gl');

export default {
  name: 'home',
  mounted() {
    // Draw map
    mapboxgl.accessToken = process.env.VUE_APP_API_KEY;
    const map = new mapboxgl.Map({
      container: 'map',
      style: 'mapbox://styles/jonathanchue/ck0zsd5qf0b8b1cn2r825i1ws',
    });
    map.addControl(new mapboxgl.NavigationControl());

    const mapData = [];

    // Get geojson and add properties
    eventData.forEach((element) => {
      const geocode = `https://api.mapbox.com/geocoding/v5/mapbox.places/${element.city}%20${element.state}.json?autocomplete=false&country=US&fuzzyMatch=false&types=place&access_token=${mapboxgl.accessToken}`;

      axios
        .get(geocode)
        .then((response) => {
          mapData.push(response.data.features[0]);
          mapData[mapData.length - 1].properties = element;
          mapData[mapData.length - 1].id = (mapData.length - 1);
        });
    });

    let hoveredStateId = null;

    map.on('load', () => {
      // Add a layer showing the places
      map.addLayer({
        id: 'places',
        type: 'circle',
        paint: {
          'circle-radius': [
            'interpolate',
            ['linear'],
            ['get', 'studentParticipation'],
            0, 5,
            500, 20,
            2000, 30,
            5000, 40,
          ],
          'circle-color': '#722229',
          'circle-stroke-width': 1.5,
          'circle-stroke-color': ['case',
            ['boolean', ['feature-state', 'hover'], false],
            '#000000',
            '#ffffff',
          ],
        },
        source: {
          type: 'geojson',
          data: {
            type: 'FeatureCollection',
            features: mapData,
          },
        },
      });

      // Create a popup, but don't add it to the map yet.
      const popup = new mapboxgl.Popup({
        closeButton: false,
        closeOnClick: false,
      });

      // When the user moves their mouse over the state-fill layer, we'll update the
      // feature state for the feature under the mouse.
      map.on('mousemove', 'places', (e) => {
        // Change the cursor style as a UI indicator.
        map.getCanvas().style.cursor = 'pointer';

        if (e.features.length > 0) {
          if (hoveredStateId) {
            map.setFeatureState({ source: 'places', id: hoveredStateId }, { hover: false });
          }
          hoveredStateId = e.features[0].id;
          map.setFeatureState({ source: 'places', id: hoveredStateId }, { hover: true });
        }

        const coordinates = e.features[0].geometry.coordinates.slice();
        const eventName = e.features[0].properties.name;

        let eventDate = e.features[0].properties.when;
        eventDate = new Date(eventDate);
        const options = { year: 'numeric', month: 'long', day: 'numeric' };
        eventDate = eventDate.toLocaleDateString('en-US', options);

        const eventCity = e.features[0].properties.city;
        const eventState = e.features[0].properties.state;
        const eventParticipation = e.features[0].properties.studentParticipation;

        // Ensure that if the map is zoomed out such that multiple
        // copies of the feature are visible, the popup appears
        // over the copy being pointed to.
        while (Math.abs(e.lngLat.lng - coordinates[0]) > 180) {
          coordinates[0] += e.lngLat.lng > coordinates[0] ? 360 : -360;
        }

        // Populate the popup and set its coordinates
        // based on the feature found.
        popup.setLngLat(coordinates)
          .setHTML(`<dl>
          <dd><strong>${eventName}</strong></dd>
          <dd>${eventDate}</dd>
          <dd>${eventCity}, ${eventState}</dd>
          <dt>Participation</dt>
          <dd>${eventParticipation}</dd>
          </dl>`)
          .addTo(map);
      });

      // When the mouse leaves the state-fill layer, update the feature state of the
      // previously hovered feature.
      map.on('mouseleave', 'places', () => {
        if (hoveredStateId) {
          map.setFeatureState({ source: 'places', id: hoveredStateId }, { hover: false });
        }
        hoveredStateId = null;

        map.getCanvas().style.cursor = '';
        popup.remove();
      });

      // Set zoom
      const bbox = [
        [-125.35917691159023, 49.26019990468981],
        [-67.73927169771598, 24.823446459736502],
      ];
      map.fitBounds(bbox, {
        padding: {
          top: 10,
          bottom: 25,
          left: 15,
          right: 5,
        },
      });
    });
  },
};
</script>

<style>
  @import url('https://api.mapbox.com/mapbox-gl-js/v1.3.1/mapbox-gl.css');

  dl {
    margin: 0;
  }
  dt {
    clear: left;
    float: left;
    font-weight: bold;
    margin-right: 5px;
  }
  dt::after {
    content: ":";
  }
  dd {
    margin: 0;
  }
  #map {
    bottom: 0;
    left: 0;
    position: absolute;
    right: 0;
    top: 0;
  }
  .mapboxgl-popup {
    font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
    min-width: 200px;
  }
  </style>
