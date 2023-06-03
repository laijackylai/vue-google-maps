<template>
  <div>
    <!-- get location -->
    <div class="location">
      <button class="input-group-text" @click="getLocation">
        Get Current Location
      </button>
      <p v-if="error">{{ error }}</p>
      <p v-else-if="currentLocation">Current Location: {{ currentLocation }}</p>
    </div>

    <!-- search -->
    <div class="input-group">
      <input
        v-model="searchQuery"
        type="text"
        class="form-control"
        placeholder="Search Location"
        @keyup.enter="searchLocation"
      />
      <button class="input-group-text" @click="searchLocation">
        <i class="bi bi-search"></i>
      </button>
    </div>

    <!-- map component -->
    <GMap
      ref="gMap"
      class="map"
      language="en"
      :center="center"
      :options="{ fullscreenControl: true }"
      :zoom="12"
    >
      <GMapMarker
        v-for="location in locations"
        :key="location.id"
        :position="{ lat: location.position.lat, lng: location.position.lng }"
      >
        <GMapInfoWindow :options="{ maxWidth: 200 }">
          <code>
            name: {{ location.name }}, lat: {{ location.position.lat }}, lng:
            {{ location.position.lng }}
          </code>
        </GMapInfoWindow>
      </GMapMarker>
    </GMap>

    <!-- locations -->
    <div>
      <table class="table">
        <thead>
          <tr>
            <th></th>
            <th>Name</th>
            <th>Latitude</th>
            <th>Longitude</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="location in locations" :key="location.id">
            <td><input v-model="location.checked" type="checkbox" /></td>
            <td
              class="clickable-cell"
              @click="
                updateMapCenter({
                  lat: location.position.lat,
                  lng: location.position.lng,
                })
              "
            >
              {{ location.name }}
            </td>
            <td>{{ location.position.lat }}</td>
            <td>{{ location.position.lng }}</td>
          </tr>
        </tbody>
      </table>
      <button v-if="locations.length > 0" @click="deleteLocations">
        Delete Selected Locations
      </button>
    </div>
  </div>
</template>

<script>
import { Loader } from '@googlemaps/js-api-loader'

export default {
  data() {
    return {
      center: {
        lat: 43.6532,
        lng: -79.3832,
      },
      locations: [
        // dummy entry
        // {
        //   checked: false,
        //   id: 0,
        //   name: 'toronto',
        //   position: {
        //     lat: 43.6532,
        //     lng: -79.383,
        //   },
        // },
      ],
      currentLocation: null,
      error: null,
      searchQuery: '',
    }
  },
  methods: {
    // functions to run at the start
    mounted() {
      this.loadGoogleMaps()
    },

    // get current location of user
    getLocation() {
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          this.handleSuccess,
          this.handleError
        )
      } else {
        this.error = 'Geolocation is not supported by this browser.'
      }
    },

    // if location is got, update map center and add new location marker
    async handleSuccess(position) {
      const latitude = position.coords.latitude
      const longitude = position.coords.longitude
      this.currentLocation = `Latitude: ${latitude}, Longitude: ${longitude}`
      const newCenter = {
        lat: latitude,
        lng: longitude,
      }
      this.updateMapCenter(newCenter)
      const newLocation = {
        checked: false,
        id: 1,
        name: 'current location',
        position: {
          lat: latitude,
          lng: longitude,
        },
      }
      await this.locations.push(newLocation)
      await this.$refs.gMap.initChildren()
    },

    // get user location error handler
    handleError(error) {
      switch (error.code) {
        case error.PERMISSION_DENIED:
          this.error = 'User denied the request for Geolocation.'
          break
        case error.POSITION_UNAVAILABLE:
          this.error = 'Location information is unavailable.'
          break
        case error.TIMEOUT:
          this.error = 'The request to get user location timed out.'
          break
        case error.UNKNOWN_ERROR:
          this.error = 'An unknown error occurred.'
          break
      }
    },

    // load google maps
    async loadGoogleMaps() {
      const loader = new Loader({
        apiKey: process.env.GOOGLE_MAPS_API_KEY,
        version: 'weekly',
        libraries: ['places'],
      })
      try {
        await loader.load()
        console.info('Google Maps API loaded successfully')
      } catch (error) {
        console.error('Error loading Google Maps API:', error)
      }
    },

    // search and get the location of the top result from the user's search query
    async searchLocation() {
      const geocoder = new google.maps.Geocoder()
      await geocoder.geocode(
        { address: this.searchQuery },
        async (results, status) => {
          if (status === 'OK') {
            if (results && results.length > 0) {
              const res = results[0]
              const id = res.place_id
              const latitude = res.geometry.location.lat()
              const longitude = res.geometry.location.lng()
              const newLocation = {
                checked: false,
                id,
                name: this.searchQuery,
                position: {
                  lat: latitude,
                  lng: longitude,
                },
              }
              await this.locations.push(newLocation)
              this.updateMapCenter({ lat: latitude, lng: longitude })
              // await this.$refs.gMap.initChildren()
            }
          } else {
            console.error(
              'Geocode was not successful for the following reason:',
              status
            )
          }
        }
      )
    },

    // delete selected locations
    async deleteLocations() {
      this.locations = this.locations.filter((location) => !location.checked)
      await this.$nextTick()
      await this.$refs.gMap.initMap()
    },

    // update map center
    updateMapCenter(newCenter) {
      this.center = newCenter
      this.$nextTick(() => {
        this.$refs.gMap.initMap()
      })
    },
  },
}
</script>

<style>
.map {
  width: 100vw;
}

.input-group {
  width: 20rem;
}

.clickable-cell {
  cursor: pointer;
}
</style>
