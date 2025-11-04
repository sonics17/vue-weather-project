<script setup>
import { onMounted, ref } from 'vue';
import WeatherWidget from './components/WeatherWidget.vue';
import Spinner from './components/Spinner.vue'

const latitude = ref(null)
const longitude = ref(null)
const isLoading = ref(false)
const errorMessage = ref(null)

function initGeolocation() {
  isLoading.value = true
  getUserPosition()
}

function successGettingGeo(position) {
  errorMessage.value = null
  longitude.value = position.coords.longitude
  latitude.value = position.coords.latitude
  isLoading.value = false
}

function handleError(error) {
  switch (error.code) {
    case GeolocationPositionError.PERMISSION_DENIED:
      errorMessage.value = 'Permission denied. Allow access to your geolocation and try again.'
      break
    case GeolocationPositionError.POSITION_UNAVAILABLE:
      errorMessage.value = 'Position unavailable. Try again later.'
      break
    default:
      errorMessage.value = 'Unknown error. Try again.'
      break
  }
}

function errorGettingGeo(error) {
  isLoading.value = false
  longitude.value = null
  latitude.value = null
  handleError(error)
}

const options = {
  enableHighAccuracy: false,
  timeout: 15000, 
  maximumAge: 30 * 60 * 1000
}

function getUserPosition() {
  navigator.geolocation.getCurrentPosition(successGettingGeo, errorGettingGeo, options)
}

onMounted(() => {
  if (!navigator.geolocation) {
    errorMessage.value = 'Geolocation is not supported by this browser.'
  }
})
</script>

<template>
  <div class="weather-app">
    <div v-if="!latitude && !longitude && !isLoading && !errorMessage" class="geo-widget widget-base widget-state--starting">
      <button class="widget__button" @click="initGeolocation">Get My Weather</button>
    </div>

    <div v-else-if="errorMessage" class="geo-widget widget-base widget-state--error">
      <div class="widget__message">{{ errorMessage }}</div>
      <button class="widget__button" @click="initGeolocation">Try again</button>
    </div>

    <WeatherWidget 
      v-else-if="longitude && latitude && !isLoading"
      :latitude="latitude"
      :longitude="longitude"
      class="geo-widget widget-base geo-widget--content"
    />

    <div v-else class="geo-widget widget-base widget-state--loading">
      <Spinner />
      <div class="widget__message">Getting your location...</div>
    </div>
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

.widget-state--starting {
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>
