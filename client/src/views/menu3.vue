<!--
  Menu3.vue
  This Vue component implements the third menu board page for a Panda Express restaurant,
  displaying appetizers and drinks with dynamic pricing and promotional content.

  Key Features:
  - Two-column layout:
    * Left: Promotional image (Apple Pie ad)
    * Right: Appetizers and Drinks sections
  - Dynamic price fetching and display
  - Size options for drinks
  - Integration with MenuBoard component
  
  Menu Sections:
  1. Appetizers (Step 4):
     - Fixed size with dynamic pricing
     - Full appetizer list from MenuBoard
  
  2. Drinks (Step 5):
     - Three size options:
       * Small with dynamic pricing
       * Medium with dynamic pricing
       * Large with dynamic pricing
     - Full drink list from MenuBoard
  
  API Integration:
  - Dynamic price fetching via axios
  - Error handling for failed requests
  - Loading state management
  
  Dependencies:
  - axios: API calls
  - Menu-Board.vue: Reusable menu component
  - applepiead.png: Promotional image asset
  - VITE_API_ENDPOINT environment variable
  
  Styling:
  - Fixed full-screen layout
  - Two-column flex design
  - Responsive image handling
  - Consistent branding colors
  - Scrollable content areas
  
  States:
  - prices: Object storing fetched price data
  
  Note: Component name 'Menu1' appears to be incorrectly set,
  should be 'Menu3' for consistency

  Author: Abhi Bhattacharyya

-->


<template>
  <div class="main-content">
    <!-- Left column for promotional image -->
    <div class="column">
      <img src="../assets/applepiead.png" alt="Get the Hot Ones Blazing Bourbon Chicken - Limited Time Offer! Panel" class="ad-image"/>
    </div>

    <!-- Right column for menu items -->
    <div class="column">
      <!-- Appetizers Section -->
      <div class="banner">
        <h1>(4) APPETIZERS</h1>
      </div>

      <!-- Appetizer Size/Price Entry -->
      <div class="entry">
        <div class="item-column">
          <h2>SMALL</h2>
        </div>    
        <div class="price-column">
          <!-- Dynamic price display with conditional rendering -->
          <span v-if="prices['Appetizer'] !== undefined" class="price">
            {{ formatPrice(prices['Appetizer']) }}
          </span>
        </div>
      </div>

      <!-- Appetizers Menu Board Component -->
      <MenuBoard :item="'Appetizers'" />

      <!-- Drinks Section -->
      <div class="banner">
        <h1>(5) DRINKS</h1>
      </div>

      <!-- Drink Size Options with Dynamic Pricing -->
      <!-- Small Drink -->
      <div class="entry">
        <div class="item-column">
          <h2>SMALL</h2>
        </div>    
        <div class="price-column">
          <span v-if="prices['Small Drink'] !== undefined" class="price">
            {{ formatPrice(prices['Small Drink']) }}
          </span>
        </div>
      </div>

      <!-- Medium Drink -->
      <div class="entry">
        <div class="item-column">
          <h2>MEDIUM</h2>
        </div>    
        <div class="price-column">
          <span v-if="prices['Medium Drink'] !== undefined" class="price">
            {{ formatPrice(prices['Medium Drink']) }}
          </span>
        </div>
      </div>

      <!-- Large Drink -->
      <div class="entry">
        <div class="item-column">
          <h2>LARGE</h2>
        </div>    
        <div class="price-column">
          <span v-if="prices['Large Drink'] !== undefined" class="price">
            {{ formatPrice(prices['Large Drink']) }}
          </span>
        </div>
      </div>

      <!-- Drinks Menu Board Component -->
      <MenuBoard :item="'Drinks'" />
    </div>   
  </div> 
</template>

<script>
import MenuBoard from '../components/Menu-Board.vue'
import axios from 'axios';

export default {
  name: 'Menu3',
  components: {
    MenuBoard,
  },
  methods: {
    // Fetches prices for all menu items from the API
    async fetchPrices() {
      try {
        const items = ['Appetizer', 'Small Drink', 'Medium Drink', 'Large Drink'];
        for (const item of items) {
          const response = await axios.get(
            import.meta.env.VITE_API_ENDPOINT + `price/${encodeURIComponent(item)}`
          );
          this.prices[item] = response.data.price;
        }
      } catch (error) {
        console.error('Error fetching prices:', error);
      }
    },
    // Formats price numbers to currency string
    formatPrice(price) {
      return price !== null ? `$${price.toFixed(2)}` : 'Loading...';
    },
  },
  
  // Initial data state
  data() {
    return {
      prices: {}, // Stores prices fetched from API
    }
  },

  // Fetch prices when component is mounted
  mounted() {
    this.fetchPrices();
  }
}
</script>

<style scoped>
/* Main container styling */
.main-content {
  position: fixed;
  top: 0;
  left: 0px;
  right: 0;
  bottom: 0;
  background-color: #000000;
  color: #e7e4d7;
  padding: 0px;
  overflow-y: auto;
  display: flex;
}

/* Two-column layout */
.column {
  flex: 1;
}

/* Menu entry styling */
.entry {
  display: flex;
  padding-top: 15px;
}

/* Item details column */
.item-column {
  padding-left: 20px;
  flex: 1;
  font-size: 1em;
}

/* Price column styling */
.price-column {
  margin: auto;
  width: 50%;
  font-size: 1.5em;
}

/* Image container styling */
.fill {
  display: flex;
  justify-content: center;
  align-items: center;
  overflow: hidden;
}

/* Promotional image styling */
.column img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

/* Section header banner styling */
.banner {
  background-color: #9b150b;
  padding: 20px;
}

/* Individual item styling */
.item {
  padding-left: 20px;
  padding: 10px;
}
</style>