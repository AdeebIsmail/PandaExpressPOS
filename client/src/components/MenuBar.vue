<!--
  MenuBar.vue
  This Vue component implements a responsive sidebar menu for a restaurant ordering system.
  It provides navigation controls, weather-based recommendations, and user authentication features.

  Key Features:
  - Dynamic menu navigation buttons for different food categories
  - Real-time weather display and weather-based food recommendations
  - User authentication status display and logout functionality
  - Accessibility features including ASL interpreter request
  - Role-based display elements (manager/cashier views)
  - Google Translate integration for language accessibility
  
  Dependencies:
  - axios: For weather API calls
  - vue-router: For navigation
  - vue-cookies: For authentication state management
  - Google Translate API: For translation services

  API Integrations:
  - Weather.gov API: Fetches local weather data for College Station, TX
  
  Author: Jensyn Huynh
-->


<template>
  <div class="menu-bar">
    <!-- Logout Button -->
    <button v-if="$cookies.get('role') == 'manager' || $cookies.get('role') == 'cashier'" @click="logout" class="logout-button">
      <span class="logout-icon">✖</span> LOGOUT
    </button>

    <h2 v-if="$cookies.get('role') == 'manager' || $cookies.get('role') == 'cashier'" class="toolbar">Signed in as: {{ capitalizedRole }} #{{$cookies.get('ID')}} </h2>

    <h2 class="menu-title">MENU</h2>
    <button class="menu-title-button" @click="$emit('selectItem', 'Appetizers')">APPETIZERS</button>
    <button class="menu-title-button" @click="$emit('selectItem', 'Bowl')">BOWL</button>
    <button class="menu-title-button" @click="$emit('selectItem', 'Plate')">PLATE</button>
    <button class="menu-title-button" @click="$emit('selectItem', 'Bigger Plate')">BIGGER PLATE</button>
    <button class="menu-title-button" @click="$emit('selectItem', 'Drinks')">DRINKS</button>
    <button class="menu-title-button" @click="$emit('selectItem', 'A La Carte')">A LA CARTE</button>
    <button class="menu-title-button" @click="showAlert">Request ASL Interpreter</button>
    <button v-if="showButton" @click="RouteToManager">
      Switch to Manager View
    </button>

    <!-- Conditional Div for Weather -->
    <div
      class="weather-message"
      v-if="
        weather &&
        weather.temperature >= 75 &&
        !(
          $cookies.get('role') == 'manager' || $cookies.get('role') == 'cashier'
        )
      "
    >
      <h2>It's hot out! Beat the heat with an ice-cold beverage:</h2>
      <component
        :is="'RecomendedItem'"
        :itemName="'Lipton Brisk Raspberry Iced Tea'"
      />
    </div>
    <div
      class="weather-message"
      v-else-if="
        weather &&
        weather.temperature < 60 &&
        !(
          $cookies.get('role') == 'manager' || $cookies.get('role') == 'cashier'
        )
      "
    >
      <h2>Feeling the fall chill? Warm up with this tasty entree:</h2>
      <component :is="'RecomendedItem'" :itemName="'Bourbon Chicken'" />
    </div>
    <div
      class="weather-message"
      v-else-if="
        weather &&
        weather.shortForecast != 'Sunny' &&
        !(
          $cookies.get('role') == 'manager' || $cookies.get('role') == 'cashier'
        )
      "
    >
      <h2>Gloomy outside? Cheer up with a fall classic:</h2>
      <component :is="'RecomendedItem'" :itemName="'Apple Pie Roll'" />
    </div>

    <Translate />
    <!-- Weather Display -->
    <div class="weather">
      <p v-if="weather">
        {{ weather.city }}, {{ weather.state }}: {{ weather.temperature }}°F,
        {{ weather.shortForecast }}
      </p>
      <p v-else>Loading weather...</p>
    </div>
  </div>
</template>

<script>
import axios from 'axios'
import Translate from '../components/translate/translateModel.vue'
import { useRouter } from 'vue-router'
import VueCookies from 'vue-cookies'
import RecomendedItem from './menu/RecommendItem.vue'

export default {
  computed: {
    capitalizedRole() {
      const role = this.$cookies.get('role');
      return role ? role.charAt(0).toUpperCase() + role.slice(1) : '';
    },
  },
  setup() {
    const router = useRouter()

    const RouteToManager = () => {
      router.push('/manager')
    }

    const logout = () => {
      // Clear user session cookies
      $cookies.remove('ID');
      $cookies.remove('role');
      // Redirect to login page
      router.push('/login');
    };

    return {
      RouteToManager, logout
    }
  },

  name: 'MenuBar',
  components: {
    Translate,
    RecomendedItem,
  },
  data() {
    return {
      weather: null,
      showButton: $cookies.get('role') == 'manager',
      hotItem: 'Bourbon Chicken',
    }
  },
  methods: {
    async fetchWeather() {
      try {
        // Coordinates of College Station
        const lat = '30.6014'
        const lon = '-96.3144'

        // Get the forecast URL
        const pointResponse = await axios.get(
          `https://api.weather.gov/points/${lat},${lon}`,
          {
            // User Agent required by weather.gov
            headers: {
              'User-Agent': 'Project3TeamCharmander (jensyn@tamu.edu)',
            },
          },
        )
        console.log('Point response:', pointResponse.data) // Check if data is returned

        const forecastUrl = pointResponse.data.properties.forecast
        const city =
          pointResponse.data.properties.relativeLocation.properties.city
        const state =
          pointResponse.data.properties.relativeLocation.properties.state

        // Fetch the forecast data
        const forecastResponse = await axios.get(forecastUrl, {
          headers: {
            'User-Agent': 'Project3TeamCharmander (jensyn@tamu.edu)',
          },
        })
        console.log('Forecast response:', forecastResponse.data) // Check if data is returned
        const forecast = forecastResponse.data.properties.periods[0] // Get current forecast

        // Update weather data
        this.weather = {
          city: city,
          state: state,
          temperature: forecast.temperature,
          shortForecast: forecast.shortForecast,
        }
      } catch (error) {
        console.error('Error fetching weather data:', error)
      }
    },
    showAlert() {
      alert('An ASL interpreter has been requested to come assist you!');
    }
  },
  mounted() {
    this.fetchWeather()
  },
  unmounted() {
    delete window.googleTranslateElementInit
  },
}
</script>

<style scoped>
.menu-bar {
  background-color: #e7e4d7;
  width: 450px;
  height: 100vh;
  padding: 20px;
  position: fixed;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-shadow: 0 4px 8px #080808;
  top: 0;
  left: 0;
  z-index: 1000;
}

.menu-title {
  margin-bottom: 0.9375em;
  font-size: 1.5em;
  text-align: center;
  color: #080808;
}

.menu-title-button{
  font-size: 1em;
}

.weather-message {
  margin-top: 10px;
  font-size: 1.5em;
  text-align: center;
  color: #080808;
}

.weather-message h2 {
  margin-bottom: 0px;
  font-size: .8em;
  text-align: center;
  color: #080808;
}

button {
  border: 2px solid #080808;
  border-radius: 30px;
  background-color: #e7e4d7;
  color: #080808;
  font: Arial;
  padding: 0.9375em;
  margin-bottom: 10px;
  cursor: pointer;
  transition:
    background-color 0.3s,
    box-shadow 0.3s;
  box-shadow: 0 4px 3px #080808;
}

button:hover {
  background-color: #d2ceb8;
}

.weather {
  margin-top: auto;
  padding: 10px;
  font-size: 1.125em;
  color: #333;
  text-align: center;
  background-color: #e7e4d7;
  border-top: 2px solid #080808;
}

.toolbar {
  margin-bottom: 35px;
  font-size: 20px;
  text-align: center;
  color: #080808;
  border-bottom: 2px solid #080808;
}

.logout-button {
  display: flex;
  align-items: center;
  justify-content: center;
  border: 2px solid #080808;
  border-radius: 30px;
  background-color: #d2ceb8;
  color: #080808;
  font: Arial;
  padding: 10px 15px;
  margin-bottom: 10px;
  cursor: pointer;
  transition: background-color 0.3s, box-shadow 0.3s;
  box-shadow: 0 4px 3px #080808;
}

.logout-button:hover {
  background-color: #938f7a;
}

.logout-icon {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-right: 10px;
  width: 20px;
  height: 20px;
  border: 2px solid #080808;
  border-radius: 50%;
  background-color: #d2ceb8;
  color: #080808;
  font-size: 14px;
  font-weight: bold;
  text-align: center;
}

</style>
