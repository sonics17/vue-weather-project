<script setup>
import { ref } from 'vue';
import WeatherWidget from './components/WeatherWidget.vue';
import Spinner from './components/Spinner.vue'

const latitude = ref(null)
const longitude = ref(null)
const isLoading = ref(false)
const errorMessage = ref(null)
const userLocationRequested = ref(false)

function successGettingGeo(position) {
  errorMessage.value = null
  longitude.value = position.coords.longitude
  latitude.value = position.coords.latitude
  isLoading.value = false
}

function handleError(error) {
  switch (error.code) {
    case GeolocationPositionError.PERMISSION_DENIED:
      errorMessage.value = 'Location access denied\nPlease allow geolocation in browser settings\nOr try IP-based location'
      break
    case GeolocationPositionError.POSITION_UNAVAILABLE:
      errorMessage.value = 'Position unavailable\nCheck your device location settings\nOr use approximate IP location'
      break
    default:
      errorMessage.value = 'Unknown error\nTry again'
      break
  }
}

function errorGettingGeo(error) {
  isLoading.value = false
  longitude.value = null
  latitude.value = null
  handleError(error)
}

async function getLocationByIP() {
  try {
    isLoading.value = true
    const response = await fetch('https://ipinfo.io/json')
    const locationData = await response.json()
    console.log(locationData)

    errorMessage.value = null

    const coords = locationData.loc.split(',')

    latitude.value = coords[0]
    longitude.value = coords[1]
  } catch (error) {
    errorMessage.value = "Can't get your geolocation"
  } finally {
    isLoading.value = false
  }
}

const options = {
  enableHighAccuracy: false,
  timeout: 15000, 
  maximumAge: 30 * 60 * 1000
}

function getUserPosition() {
  navigator.geolocation.getCurrentPosition(successGettingGeo, errorGettingGeo, options)
}

function startGettingGeolocation() {
  userLocationRequested.value = true
  isLoading.value = true
  errorMessage.value = null
  
  if (navigator.geolocation) {
    getUserPosition()
  } else {
    errorMessage.value = 'Geolocation is not supported by this browser.'
    isLoading.value = false
  }
}

</script>

<template>
  <div class="weather-app">
    <div v-if="!userLocationRequested && !latitude && !longitude && !errorMessage" 
         class="geo-widget widget-base widget-state--starting">
      <button class="widget__button" @click="startGettingGeolocation">Get my weather</button>
    </div>


    <div v-else-if="userLocationRequested && isLoading" class="geo-widget widget-base widget-state--loading">
      <Spinner />
      <div class="widget__message">Getting your location...</div>
    </div>

    <div v-else-if="errorMessage && userLocationRequested" class="geo-widget widget-base widget-state--error">
      <div class="widget__message">{{ errorMessage }}</div>
      <div class="widget__buttons">
        <button class="widget__button" @click="getUserPosition">Try again</button>
        <button class="widget__button" @click="getLocationByIP">Use approximate location</button>
      </div>
    </div>

    <WeatherWidget 
      v-else-if="userLocationRequested && longitude && latitude && !isLoading"
      :latitude="Number(latitude)"
      :longitude="Number(longitude)"
      class="geo-widget widget-base geo-widget--content"
    />
  </div>
</template>

<style scoped>
.weather-app {
  box-sizing: border-box;
  min-height: 100dvh;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1rem;
}
</style>
