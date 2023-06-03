<template>
  <div>
    <div class="container">
      <!-- get location -->
      <div class="location">
        <button class="input-group-text" @click="getLocation">
          Get Current Location
        </button>
        <p v-if="error" class="error">{{ error }}</p>
        <p v-else-if="currentLocation" class="current-location">
          {{ currentLocation }}
        </p>
      </div>

      <!-- search -->
      <div class="input-group">
        <input
          id="autocomplete-input"
          v-model="searchQuery"
          type="text"
          class="form-control"
          placeholder="Search Location"
          @keyup.enter="searchLocation"
          @input="handleInputChange"
        />
        <button class="input-group-text" @click="searchLocation">
          <i class="bi bi-search"></i>
        </button>
      </div>
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
    <div class="info">
      <div class="table-top-row">
        <ul class="pagination">
          <li class="page-item" :class="{ disabled: currentPage === 1 }">
            <a class="page-link" href="#" @click="currentPage -= 1">Previous</a>
          </li>
          <li
            class="page-item"
            :class="{ active: page === currentPage }"
            v-for="page in totalPages"
            :key="page"
          >
            <a class="page-link" @click="currentPage = page" href="#">{{
              page
            }}</a>
          </li>
          <li
            class="page-item"
            :class="{ disabled: currentPage === totalPages }"
          >
            <a class="page-link" href="#" @click="currentPage += 1">Next</a>
          </li>
        </ul>
        <button v-if="locations.length > 0" @click="deleteLocations">
          Delete Selected Locations
        </button>
      </div>
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
          <tr v-for="location in displayedLocations" :key="location.id">
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
      currentPage: 1,
      pageSize: 10,
      currentLocation: null,
      error: null,
      searchQuery: '',
    }
  },
  computed: {
    startIndex() {
      return (this.currentPage - 1) * this.pageSize
    },
    endIndex() {
      return this.startIndex + this.pageSize - 1
    },
    displayedLocations() {
      return this.locations.slice(this.startIndex, this.endIndex + 1)
    },
    totalPages() {
      const res = Math.ceil(this.locations.length / this.pageSize)
      if (res === 0) {
        return 1
      } else {
        return res
      }
    },
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
      if (!this.checkDuplicate(newLocation)) {
        await this.locations.push(newLocation)
      }
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

    // load google maps api
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
      try {
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
                if (!this.checkDuplicate(newLocation)) {
                  await this.locations.push(newLocation)
                }
                this.updateMapCenter({ lat: latitude, lng: longitude })
              }
            } else {
              console.error('No locations found:', status)
              alert('Location query failed: ' + status)
            }
          }
        )
      } catch (error) {
        console.error(error)
      }
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

    // check for duplicate location
    checkDuplicate(newLocation) {
      // return false
      return this.locations.some((item) => item.id === newLocation.id)
    },

    // autocomplete feature
    handleInputChange() {
      console.log(google.maps)
      console.log(google.maps.places)
      // const input = document.getElementById('autocomplete-input')
      // const autocomplete = new google.maps.places.Autocomplete(input)
      // autocomplete.addListener('place_changed', () => {
      //   const place = this.autocomplete.getPlace()
      //   console.log(place)
      // })
    },

    // go to next page
    nextPage() {
      if (this.currentPage < this.totalPages) {
        this.currentPage++
      }
    },

    // go to previous page
    previousPage() {
      if (this.currentPage > 1) {
        this.currentPage--
      }
    },
  },
}
</script>

<style>
.container {
  display: flex;
  flex-direction: column;
  padding: 1rem;
}

.location {
  display: flex;
  flex-direction: row;
  gap: 1rem;
  align-items: center;
}

.error,
.current-location {
  margin: 0;
}

.map {
  width: 100vw;
}

.input-group {
  width: 20rem;
}

.clickable-cell {
  cursor: pointer;
}

.info {
  padding: 1rem;
}

.table-top-row {
  display: flex;
  flex-direction: row-reverse;
  justify-content: space-between;
  align-items: center;
}
</style>
