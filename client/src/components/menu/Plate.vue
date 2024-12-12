<!--
  Plate.vue
  This Vue component handles logic for displaying and selecting items from the Panda Express plate menu


  Key Features:
  - display available sides and entrees
  - ensure user selects one side and two entrees
  - adjust pricing based on premium items
  - allow user to add item to cart
  - show premium/out of stock items when applicable
  
  Props:
  - None
  
  Emits:
  - addToCart: add item to cart display
  - addToTrasactionCart: add item to the current transaction for data tracking
  
  Assets Required:
  - @/assets/star.png
  - @/assets/"itemname".png
  
  Accessibility:
  - Keyboard navigation support
  - Clear visual indicators
  - Proper focus management
  - Semantic HTML structure
-->
<template>
  <div class="plate">
    <h2>Pick 1 Side</h2>
    <div class="grid">
      <button
        v-for="side in sides"
        :key="side"
        @click="selectSide(side)"
        :disabled="isOutOfStock(side, 'Side')"
        :class="{
          selected: selectedSide === side,
          'out-of-stock': isOutOfStock(side, 'Side'),
        }"
      >
        <img
          v-if="getSideImage(side)"
          :src="getSideImage(side)"
          :alt="'Digital image of ' + getSideName(side) + ' as presented on the Panda Express menu'"
          class="side-image"
          @error="handleImageError"
        />
        <span>{{ getSideName(side) }}</span>
        <span v-if="isOutOfStock(side, 'Side')" class="out-of-stock-label"
          >Out of Stock</span
        >
        <span v-if="selectedSide === side" class="checkmark">✓</span>
      </button>
    </div>

    <h2>Pick 2 Entrees</h2>
    <div class="grid">
      <button
        v-for="entree in entrees"
        :key="entree"
        @click="selectEntree(entree)"
        :disabled="isOutOfStock(entree, 'Entree')"
        :class="{
          selected: selectedEntrees.includes(entree),
          'out-of-stock': isOutOfStock(entree, 'Entree'),
        }"
      >
        <img
          v-if="getEntreeImage(entree)"
          :src="getEntreeImage(entree)"
          :alt="'Digital image of the' + getEntreeName(side) + ' entree as presented on the Panda Express menu'"
          class="entree-image"
          @error="handleImageError"
        />
        <span>{{ getEntreeName(entree) }}</span>
        <span v-if="isOutOfStock(entree, 'Entree')" class="out-of-stock-label"
          >Out of Stock</span
        >
        <span v-if="selectedEntrees.includes(entree)" class="checkmark">✓</span>
        <span class="premium-label-container">
          <img
            v-if="isPremium(entree)"
            src="/src/assets/star.png"
            alt="Premium"
            class="star-icon"
          />
          <span class="premium-label">Premium Item + $1.50</span>
        </span>
      </button>
    </div>

    <!-- Add to Cart Button -->
    <button class="add-to-cart" @click="addToCart" :disabled="!canAddToCart">
      Add to Cart
    </button>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'Plate',
  props: {
    outOfStockItems: {
      type: Object,
      default: () => ({}), // Out-of-stock data passed from MainContent.vue
    },
  },
  data() {
    return {
      sides: [],
      entrees: [],
      price: null,
      selectedSide: null,
      selectedEntrees: [], // Track selected entrees as an array
      premiumEntrees: [], // Add premium entrees
    }
  },
  computed: {
    canAddToCart() {
      // Can only add to cart if 1 side and exactly 2 entrees are selected
      return this.selectedSide && this.selectedEntrees.length === 2
    },
  },
  methods: {
    async fetchMenuItems() {
      try {
        const sideResponse = await axios.get(
          import.meta.env.VITE_API_ENDPOINT + 'menu/Side',
        )
        const entreeResponse = await axios.get(
          import.meta.env.VITE_API_ENDPOINT + 'menu/Entree',
        )
        this.sides = sideResponse.data
        this.entrees = entreeResponse.data
      } catch (error) {
        console.error('Error fetching menu items:', error)
      }
    },
    async fetchPrice() {
      try {
        const response = await axios.get(
          import.meta.env.VITE_API_ENDPOINT + 'price/Plate',
        )
        this.price = response.data.price // Assign the price
      } catch (error) {
        console.error('Error fetching price:', error)
      }
    },
    isOutOfStock(item, category) {
      return this.outOfStockItems[category]?.includes(
        this.getSideName(item) || this.getEntreeName(item),
      )
    },
    selectSide(side) {
      // Select one side and deselect the others
      this.selectedSide = this.selectedSide === side ? null : side
    },
    selectEntree(entree) {
      // Add or remove entrees, ensuring only two can be selected
      if (this.selectedEntrees.includes(entree)) {
        this.selectedEntrees = this.selectedEntrees.filter(
          item => item !== entree,
        )
      } else if (this.selectedEntrees.length < 2) {
        this.selectedEntrees.push(entree)
      }
    },
    async checkPremiumStatus() {
      try {
        // Remove trailing slash from the endpoint if it exists
        const apiUrl = import.meta.env.VITE_API_ENDPOINT.replace(/\/$/, '') // Ensures no trailing slash
        const url = `${apiUrl}/ispremium/`

        console.log(`Requesting URL: ${url}`) // Log the constructed URL

        const response = await axios.get(url)
        console.log(response.data) // Log the response
        console.log('hello')
        const isPremium = response.data.premium
        for (let i = 0; i < response.data.length; i++) {
          console.log(response.data[i])
          this.premiumEntrees.push(response.data[i].name)
        }
      } catch (error) {
        console.error(
          'Error checking premium status:',
          error.response ? error.response.data : error.message,
        )
      }
    },
    isPremium(entree) {
      return this.premiumEntrees.includes(this.getEntreeName(entree)) // Check if the entree is in the premium list
    },

    addToCart() {
      if (this.canAddToCart) {
        // Fetch the base item ID for "Plate" dynamically
        axios
          .get(import.meta.env.VITE_API_ENDPOINT + `itemid/Plate`)
          .then(response => {
            const baseItemID = response.data.itemID // Dynamically fetched base item ID
            let price = this.price
            let premiumCount = 0

            // Check if any of the selected entrees are premium
            this.selectedEntrees.forEach(entree => {
              if (this.isPremium(entree)) {
                premiumCount++
              }
            })

            // Add $1.50 for each premium entree
            price += premiumCount * 1.5

            // Get the side and entree names
            const sideName = this.getSideName(this.selectedSide)
            const entreeNames = this.selectedEntrees.map(entree =>
              this.getEntreeName(entree),
            )

            // Fetch foodIDs for the selected side and entrees
            Promise.all([
              axios.get(
                import.meta.env.VITE_API_ENDPOINT +
                  `foodid/${encodeURIComponent(sideName)}`,
              ),
              ...this.selectedEntrees.map(entree =>
                axios.get(
                  import.meta.env.VITE_API_ENDPOINT +
                    `foodid/${encodeURIComponent(this.getEntreeName(entree))}`,
                ),
              ),
            ])
              .then(([sideResponse, entreeResponse1, entreeResponse2]) => {
                const sideFoodID = sideResponse.data.foodID
                const entreeFoodID1 = entreeResponse1.data.foodID
                const entreeFoodID2 = entreeResponse2.data.foodID

                // Construct the transactionEntry array
                const transactionEntry = [
                  baseItemID,
                  sideFoodID,
                  entreeFoodID1,
                  entreeFoodID2,
                  0,
                ]

                // Construct the item to emit
                const item = {
                  name: `Plate (${sideName} + ${entreeNames.join(', ')})`,
                  price: price,
                  transactionEntry: transactionEntry, // Include for further use
                  isPremium: premiumCount > 0,
                }

                this.$emit('addToCart', item) // Emit the item to the parent
                this.$emit('addToTransactionCart', transactionEntry)
                console.log('Transaction Added:', transactionEntry)

                // Reset selections
                this.selectedSide = null
                this.selectedEntrees = []
              })
              .catch(error => {
                console.error('Error fetching food IDs:', error)
                alert('Cannot add to transaction cart.')
              })
          })
          .catch(error => {
            console.error('Error fetching base item ID:', error)
            alert('Cannot fetch base item ID.')
          })
      }
    },
    getSideName(side) {
      if (typeof side === 'string') {
        return side
      } else if (side && side.name) {
        return side.name
      }
      return 'Unknown Side'
    },
    getSideImage(side) {
      let name = this.getSideName(side)
      if (!name) return null
      const fileName = `${name.toLowerCase().replace(/\s+/g, '')}.png`
      const imagePath = `/src/assets/${fileName}`
      return new URL(`/src/assets/${fileName}`, import.meta.url).href
    },
    getEntreeName(entree) {
      if (typeof entree === 'string') {
        return entree
      } else if (entree && entree.name) {
        return entree.name
      }
      return 'Unknown Entree'
    },
    getEntreeImage(entree) {
      let name = this.getEntreeName(entree)
      if (!name) return null
      const fileName = `${name.toLowerCase().replace(/\s+/g, '')}.png`
      const imagePath = `/src/assets/${fileName}`
      return new URL(`/src/assets/${fileName}`, import.meta.url).href
    },
    handleImageError(event) {
      console.error('Image failed to load:', event.target.src)
      event.target.style.display = 'none'
    },
  },
  mounted() {
    this.fetchMenuItems() // Fetch menu items when the component mounts
    this.fetchPrice()
    this.checkPremiumStatus()
  },
}
</script>

<style scoped>
.plate {
  padding: 20px;
}

.grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 10px;
  margin-bottom: 20px;
}

button {
  border: 2px solid black;
  border-radius: 10px;
  background-color: #e7e4d7;
  color: black;
  padding: 10px;
  cursor: pointer;
  transition:
    background-color 0.3s,
    box-shadow 0.3s;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  height: auto;
  /* min-height: 150px; */
  position: relative;
}

button:hover {
  background-color: #d2ceb8;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.selected {
  background-color: #d2ceb8;
  box-shadow: 0 0 0 2px #080808;
}

.checkmark {
  position: absolute;
  top: 5px;
  right: 5px;
  color: green;
  font-size: 20px;
  font-weight: bold;
}

.side-image,
.entree-image {
  width: 150px;
  height: 150px;
  object-fit: contain;
  margin-bottom: 5px;
}

.add-to-cart-container {
  max-height: min-content;
  position: fixed;
  box-shadow: 0 4px 3px #080808;
}

.add-to-cart {
  padding: 0.9375em 0.9375em;
  font-size: 0.9375em;
  background-color: #4caf50;
  color: rgb(0, 0, 0);
  border: none;
  border-radius: 10px;
  position: fixed;
  top: 45px;
  right: 145px;
  z-index: 1000;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition:
    background-color 0.3s,
    box-shadow 0.3s;
  height: 30px;
  box-shadow: 0 4px 3px #080808;
}

.add-to-cart:disabled {
  background-color: #e7e4d7;
  box-shadow: 0 4px 3px #080808;
  cursor: not-allowed;
}

.grid button {
  margin: 5px;
  padding: 10px;
  font-size: 1em
}


.star-icon {
  width: 30px;
  height: 30px;
  /* position: absolute; */
  /* top: 1px;
    left: 3px;
    /* Ensure it's above the content */
  /* z-index: 1; */
}

.premium-label-container {
  position: absolute;
  /* Keep it absolute to retain original positioning */
  top: 5px;
  /* Adjust as per your original design */
  left: 5px;
  /* Adjust as per your original design */
  z-index: 1;
  /* Ensure it's above other content */
  display: flex;
  /* Align content inside properly */
  align-items: center;
  /* Center the icon and label vertically */
  justify-content: center;
  /* Center the icon and label horizontally */
}

.premium-label {
  visibility: hidden;
  background-color: black;
  color: white;
  text-align: center;
  border-radius: 5px;
  padding: 5px;
  position: absolute;
  bottom: 30px;
  /* Reduced distance to bring it closer to the star icon */
  left: 50%;
  transform: translateX(-50%);
  white-space: nowrap;
  z-index: 10;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.3);
  font-size: 12px;
}

.premium-label-container:hover .premium-label {
  visibility: visible;
}

.out-of-stock {
  background-color: #ccc;
  pointer-events: none;
  opacity: 0.6;
}

.out-of-stock-label {
  color: red;
  font-size: 0.9em;
  font-weight: bold;
}
</style>
