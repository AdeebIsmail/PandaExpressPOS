<!--
  Drinks.vue
  This Vue component handles logic for displaying and selecting items from the Panda Express drink menu


  Key Features:
  - display available drinks
  - allow for size selection
  - adjust pricing based on size
  - allow user to add items to cart 
  - show premium/out of stock items when applicable
  
  Props:
  - None
  
  Emits:
  - addToCart: add item to cart display
  - addToTrasactionCart: add item to the current transaction for data tracking
  
  Assets Required:
  - @/assets/"itemname".png
  
  Accessibility:
  - Keyboard navigation support
  - Clear visual indicators
  - Proper focus management
  - Semantic HTML structure
-->

<template>
  <div class="drink">
    <h2 v-if="showHeader">Pick 1 or more Drinks</h2>
    <div class="grid">
      <button
        v-for="drink in drinks"
        :key="drink"
        ref="drinkButtons"
        @click="toggleDrinks(drink)"
        :disabled="isOutOfStock(drink)"
        :class="{
          selected: isSelected(drink),
          'out-of-stock': isOutOfStock(drink),
        }"
      >
        <img
          v-if="getDrinkImage(drink)"
          :src="getDrinkImage(drink)"
          :alt="'Digital image of the' + getDrinkName(drink) + ' drink as presented on the Panda Express menu'"
          class="drink-image"
          @error="handleImageError"
        />
        <span>{{ getDrinkName(drink) }}</span>
        <span v-if="isOutOfStock(drink)" class="out-of-stock-label"
          >Out of Stock</span
        >
        <span v-if="isSelected(drink)" class="checkmark">✓</span>
        <span v-if="getSelectedSize(drink)" class="size-tag">
          {{ getSelectedSize(drink) }}
        </span>
      </button>
    </div>
    <!-- Size Selection Modal -->
    <div v-if="showSizeModal && showHeader" class="size-modal">
      <h3 class="modal-title">
        Select Size for {{ getDrinkName(currentItem) }}
      </h3>
      <div>
        <button
          v-for="size in sizeOptions.drink.filter(
            size =>
              size.name !== 'Aquafina' && size.name !== 'Gatorade Lemon Lime',
          )"
          :key="size.name"
          ref="sizeButtons"
          @click="selectSize(size)"
          @keydown.enter="selectSize(size)"
          @keydown.space="selectSize(size)"
          tabindex="0"
        >
          {{ size.name }} - ${{
            size.price ? size.price.toFixed(2) : 'Loading...'
          }}
        </button>
      </div>
      <!-- Red X Button -->
      <button
        @click="cancelSizeSelection"
        @keydown.enter="cancelSizeSelection"
        @keydown.space="cancelSizeSelection"
        tabindex="0"
        class="close-button"
      >
        ✖
      </button>
    </div>
    <!-- Add to Cart Button -->
    <button
      v-if="showHeader"
      class="add-to-cart"
      @click="addToCart"
      :disabled="!canAddToCart"
    >
      Add to Cart
    </button>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'Drink',
  props: {
    showHeader: {
      type: Boolean,
      default: true,
    },
    outOfStockItems: {
      type: Object,
      default: () => ({}), // Out-of-stock data passed from MainContent.vue
    },
  },
  data() {
    return {
      drinks: [],
      selectedDrinks: [],
      showSizeModal: false,
      currentItem: null,
      currentItemType: null,
      sizeOptions: {
        drink: [
          { name: 'Small', price: null, baseItemID: null },
          { name: 'Medium', price: null, baseItemID: null },
          { name: 'Large', price: null, baseItemID: null },
        ],
      },
      transactionEntries: [], // Track the transaction entries to be pushed to the database
    }
  },
  computed: {
    canAddToCart() {
      return this.selectedDrinks.length > 0
    },
  },
  methods: {
    async fetchMenuItems() {
      try {
        const drinkResponse = await axios.get(
          import.meta.env.VITE_API_ENDPOINT + 'menu/Drink',
        )
        this.drinks = drinkResponse.data
      } catch (error) {
        console.error('Error fetching menu items:', error)
      }
    },
    async fetchItemPrices() {
      try {
        // Fetch prices and itemIDs for regular drinks with sizes
        const sizeType = ['drink']
        for (const type of sizeType) {
          const sizes = this.sizeOptions[type]

          for (const size of sizes) {
            const itemName = `${size.name} ${type.charAt(0).toUpperCase() + type.slice(1)}`
            const priceResponse = await axios.get(
              import.meta.env.VITE_API_ENDPOINT +
                `price/${encodeURIComponent(itemName)}`,
            )
            const idResponse = await axios.get(
              import.meta.env.VITE_API_ENDPOINT +
                `itemid/${encodeURIComponent(size.name + ' Drink')}`,
            )
            size.price = priceResponse.data.price // Assign price directly to the size object
            size.baseItemID = idResponse.data.itemID
          }
        }

        // Fetch prices and itemIDs for bottled drinks
        const bottledDrinks = ['Aquafina', 'Gatorade Lemon Lime']
        for (const drink of bottledDrinks) {
          const priceResponse = await axios.get(
            import.meta.env.VITE_API_ENDPOINT +
              `price/${encodeURIComponent(drink)}`,
          )
          const idResponse = await axios.get(
            import.meta.env.VITE_API_ENDPOINT +
              `itemid/${encodeURIComponent(drink)}`,
          )
          this.sizeOptions.drink.push({
            name: drink,
            price: priceResponse.data.price,
            baseItemID: idResponse.data.itemID,
          })
        }
      } catch (error) {
        console.error('Error fetching item prices:', error)
      }
    },
    async toggleDrinks(drink) {
      const drinkName = this.getDrinkName(drink)

      if (
        ['aquafina', 'gatorade lemon lime'].includes(drinkName.toLowerCase())
      ) {
        // Add bottled drinks directly with their specific prices
        const bottledDrink = this.sizeOptions.drink.find(
          option => option.name.toLowerCase() === drinkName.toLowerCase(),
        )
        if (bottledDrink) {
          const itemToAdd = {
            name: `Drink: ${drinkName}`,
            price: bottledDrink.price,
          }

          const drinkID = await this.getDrinkID(drink) // Fetch the drinkID for bottled drinks
          this.transactionEntries.push([
            bottledDrink.baseItemID,
            drinkID,
            0,
            0,
            0,
          ]) // Add transaction entry

          this.selectedDrinks.push(itemToAdd)
        } else {
          console.error('Bottled drink not found in sizeOptions:', drinkName)
        }
      } else {
        // Show size modal for regular drinks
        this.currentItem = drink
        this.currentItemType = 'drink'
        this.showSizeModal = true
      }

      this.$nextTick(() => {
        // Focus on the first size option
        const firstSizeButton = this.$refs.sizeButtons?.[0]
        if (firstSizeButton) {
          firstSizeButton.focus()
        }
      })
    },
    isOutOfStock(drink) {
      const drinkName = this.getDrinkName(drink)
      return this.outOfStockItems.Drink?.includes(drinkName) || false
    },
    isSelected(drink) {
      const drinkName = this.getDrinkName(drink)
      return this.selectedDrinks.some(
        selectedDrink =>
          selectedDrink.name === `Drink: ${drinkName}` ||
          selectedDrink.name.startsWith(`Drink: ${drinkName} (`),
      )
    },
    getSelectedSize(drink) {
      const selectedDrink = this.selectedDrinks.find(item =>
        item.name.startsWith(`Drink: ${this.getDrinkName(drink)}`),
      )
      return selectedDrink ? selectedDrink.size : null
    },
    async selectSize(size) {
      const selectedDrink = this.getDrinkName(this.currentItem) // Get the drink name (for logging/display)
      const baseItemID = size.baseItemID // Get the baseItemID for the size
      const drinkID = await this.getDrinkID(this.currentItem) // Fetch the correct drinkID using getDrinkID

      const itemToAdd = {
        name: `Drink: ${selectedDrink} (${size.name})`,
        price: size.price,
        size: size.name,
        baseItemID: baseItemID,
      }

      // Remove any existing selection for this drink
      this.selectedDrinks = this.selectedDrinks.filter(
        item => !item.name.startsWith(`Drink: ${selectedDrink}`),
      )
      this.selectedDrinks.push(itemToAdd)

      // Add to transaction entries with the correct drinkID
      this.transactionEntries.push([baseItemID, drinkID, 0, 0, 0])

      // Close size selection modal
      this.showSizeModal = false
      this.currentItem = null

      this.$nextTick(() => {
        const drinkButton = this.$refs.drinkButtons?.[this.currentItem]
        if (drinkButton) {
          drinkButton.focus()
        }
      })
    },
    cancelSizeSelection() {
      this.showSizeModal = false
      this.currentItem = null
    },
    addToCart() {
      if (this.canAddToCart) {
        // Emit the selected drinks array to the cart
        this.$emit('addToCart', this.selectedDrinks)
        this.$emit('addToTransactionCart', this.transactionEntries[0])
        console.log('Transaction Added: ', this.transactionEntries[0])

        // Clear selections after adding to cart
        this.selectedDrinks = []
        this.transactionEntries = [] // Clear transaction entries
      }
    },
    getDrinkName(drink) {
      if (typeof drink === 'string') {
        return drink
      } else if (drink && drink.name) {
        return drink.name
      }

      return 'Unknown Drink'
    },
    async getDrinkID(drink) {
      try {
        // Fetch drinkID for a given drink name from the database
        const drinkName = this.getDrinkName(drink)
        const response = await axios.get(
          import.meta.env.VITE_API_ENDPOINT +
            `foodid/${encodeURIComponent(drinkName)}`,
        )

        if (response.data && response.data.foodID) {
          return response.data.foodID // Return the resolved foodID
        } else {
          throw new Error(`No foodID found for drink: ${drinkName}`)
        }
      } catch (error) {
        console.error(`Error fetching foodID for drink "${drinkName}":`, error)
        throw error // Re-throw to allow handling in calling code
      }
    },
    getDrinkImage(drink) {
      let name = this.getDrinkName(drink)
      if (!name) return null
      const fileName = `${name.toLowerCase().replace(/\s+/g, '')}.png`
      return new URL(`/src/assets/${fileName}`, import.meta.url).href
    },
  },
  mounted() {
    this.fetchMenuItems() // Fetch menu items when the component mounts
    this.fetchItemPrices()
  },
}
</script>

<style scoped>
.drink {
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

.size-modal button {
  margin: 5px;
  padding: 10px;
  font-size: 1em;
}

.grid button {
  margin: 5px;
  padding: 10px;
  font-size: 1em;
}

button:hover {
  background-color: #d2ceb8;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.size-modal button:focus {
  outline: 2px solid #007bff; /* Blue outline */
  outline-offset: 1px;
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

.drink-image {
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

.size-modal {
  position: fixed;
  top: 50%;
  left: 60%;
  transform: translate(-50%, -50%);
  background-color: white;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.3);
  z-index: 1001;
  color: #080808;
}

.size-modal button {
  margin: 5px;
  padding: 10px;
}

.close-button {
  position: absolute;
  /* Positioning relative to the modal */
  bottom: 10px;
  /* Adjust as needed */
  right: 10px;
  /* Adjust as needed */
  background-color: transparent;
  /* No background */
  border: none;
  /* No border */
  color: red;
  /* Red color for the X */
  font-size: 1.5em;
  /* Adjust font size as needed */
  cursor: pointer;
  /* Change cursor to pointer */
}

.close-button:hover {
  color: darkred;
  /* Change color on hover for better UX */
  background-color: transparent !important;
  box-shadow: none !important;
}

.size-tag {
  margin-left: 5px;
  padding: 2px 6px;
  background-color: #4caf50;
  color: white;
  font-size: 12px;
  border-radius: 12px;
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
