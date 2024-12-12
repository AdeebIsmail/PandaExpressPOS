<!--
  ALaCarte.vue
  This Vue component handles logic for displaying and selecting items from the Panda Express A la Carte menu


  Key Features:
  - display sides and entrees
  - allow for size selection
  - allow user to add iteams to cart 
  - show premium/out of stock items when applicable
  
  Props:
  - None
  
  Emits:
  - addToCart: add item to cart display
  - addToTrasactionCart: add item to the current transaction for data tracking
  
  State:
  - selectedPayment: Currently selected payment method
  - orderConfirmed: Order confirmation status
  - orderDetails: Transaction and timing information
  
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
  <div class="alacarte">
    <h2>Pick 1 or more Sides</h2>
    <div class="grid">
      <button
        v-for="side in sides"
        :key="side.name"
        ref="sideButtons"
        @click="toggleSides(side)"
        :disabled="isOutOfStock(side, 'Side')"
        :class="{ 'out-of-stock': isOutOfStock(side, 'Side') }"
      >
        <img
          v-if="getSideImage(side)"
          :src="getSideImage(side)"
          :alt="'Digital image of ' + getSideName(side) + ' as presented on the Panda Express menu'"
          class="side-image"
          @error="handleImageError"
        />
        <span>{{ getSideName(side) }}</span>
        <span v-if="isSelected(side, 'side')" class="checkmark">✓</span>
        <span v-if="getSelectedSize(side, 'side')" class="size-tag">
          {{ getSelectedSize(side, 'side') }}
        </span>
        <span v-if="isOutOfStock(side, 'Side')" class="out-of-stock-label"
          >Out of Stock</span
        >
      </button>
    </div>

    <h2>Pick 1 or more Entrees</h2>
    <div class="grid">
      <button
        v-for="entree in entrees"
        :key="entree.name"
        ref="entreeButtons"
        @click="toggleEntrees(entree)"
        :disabled="isOutOfStock(entree, 'Entree')"
        :class="{ 'out-of-stock': isOutOfStock(entree, 'Entree') }"
      >
        <img
          v-if="getEntreeImage(entree)"
          :src="getEntreeImage(entree)"
          :alt="'Digital image of the' + getEntreeName(side) + ' entree as presented on the Panda Express menu'"
          class="entree-image"
          @error="handleImageError"
        />
        <span>{{ getEntreeName(entree) }}</span>
        <span v-if="isSelected(entree, 'entree')" class="checkmark">✓</span>
        <span v-if="getSelectedSize(entree, 'entree')" class="size-tag">
          {{ getSelectedSize(entree, 'entree') }}
        </span>
        <span v-if="isOutOfStock(entree, 'Entree')" class="out-of-stock-label"
          >Out of Stock</span
        >

        <span v-if="isPremium(entree)" class="premium-label-container">
          <img src="/src/assets/star.png" alt="Premium" class="star-icon" />
          <span class="premium-label">Premium Item + $1.50</span>
        </span>
      </button>
    </div>

    <!-- Size Selection Modal -->
    <div v-if="showSizeModal" class="size-modal">
      <h3 class="modal-title">
        Select Size for
        {{
          currentItemType === 'side'
            ? getSideName(currentItem)
            : getEntreeName(currentItem)
        }}
      </h3>
      <div v-if="currentItemType === 'side'">
        <button
          v-for="size in sizeOptions.side"
          :key="size.name"
          ref="side_sizeButtons"
          @click="selectSize(size, 'side')"
          @keydown.enter="selectSize(size, 'side')"
          @keydown.space="selectSize(size, 'side')"
          tabindex="0"
        >
          {{ size.name }} - ${{
            size.price ? size.price.toFixed(2) : 'Loading...'
          }}
        </button>
      </div>
      <div v-else>
        <button
          v-for="size in sizeOptions.entree"
          :key="size.name"
          ref="entree_sizeButtons"
          @click="selectSize(size, 'entree')"
          @keydown.enter="selectSize(size, 'entree')"
          @keydown.space="selectSize(size, 'entree')"
          tabindex="0"
        >
          {{ size.name }} - ${{
            size.price ? size.price.toFixed(2) : 'Loading...'
          }}
        </button>
      </div>

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
    <button class="add-to-cart" @click="addToCart" :disabled="!canAddToCart">
      Add to Cart
    </button>
  </div>
</template>

<script>
import axios from 'axios'

export default {
  name: 'ALaCarte',
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
      selectedSides: [],
      selectedEntrees: [],
      showSizeModal: false,
      currentItem: null,
      currentItemType: null,
      sizeOptions: {
        side: [
          { name: 'Medium', price: null, baseItemID: null },
          { name: 'Large', price: null, baseItemID: null },
        ],
        entree: [
          { name: 'Small', price: null, baseItemID: null },
          { name: 'Medium', price: null, baseItemID: null },
          { name: 'Large', price: null, baseItemID: null },
        ],
      },
      premiumEntrees: [], // Add premium entrees
      premiumPrices: {
        // Define premium item prices by size

        Small: 6.7,
        Medium: 11.5,
        Large: 15.7,
      },
    }
  },
  computed: {
    canAddToCart() {
      return this.selectedSides.length > 0 || this.selectedEntrees.length > 0
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

        // Fetch base item IDs for sizes
        for (const type of ['side', 'entree']) {
          for (const size of this.sizeOptions[type]) {
            const response = await axios.get(
              import.meta.env.VITE_API_ENDPOINT +
                `itemid/${encodeURIComponent(size.name + ' ' + type.charAt(0).toUpperCase() + type.slice(1))}`,
            )
            size.baseItemID = response.data.itemID
          }
        }
      } catch (error) {
        console.error('Error fetching menu items:', error)
      }
    },
    async fetchItemPrices() {
      try {
        const sizeTypes = ['side', 'entree']
        for (const type of sizeTypes) {
          const sizes = this.sizeOptions[type]

          for (const size of sizes) {
            const itemName = `${size.name} ${type.charAt(0).toUpperCase() + type.slice(1)}`
            const response = await axios.get(
              import.meta.env.VITE_API_ENDPOINT +
                `price/${encodeURIComponent(itemName)}`,
            )
            size.price = response.data.price // Assign price directly to the size object
          }
        }
      } catch (error) {
        console.error('Error fetching item prices:', error)
      }
    },
    async getFoodID(item) {
      try {
        const itemName =
          this.currentItemType === 'side'
            ? this.getSideName(item)
            : this.getEntreeName(item)
        const response = await axios.get(
          import.meta.env.VITE_API_ENDPOINT +
            `foodid/${encodeURIComponent(itemName)}`,
        )
        return response.data.foodID
      } catch (error) {
        console.error('Error fetching foodID for item:', error)
        throw error
      }
    },
    isOutOfStock(item, category) {
      return this.outOfStockItems[category]?.includes(
        this.getSideName(item) || this.getEntreeName(item),
      )
    },
    toggleSides(side) {
      this.currentItem = side
      this.currentItemType = 'side'
      this.showSizeModal = true

      this.$nextTick(() => {
        // Focus on the first size option
        const firstSizeButton = this.$refs.side_sizeButtons?.[0]
        if (firstSizeButton) {
          firstSizeButton.focus()
        }
      })
    },
    toggleEntrees(entree) {
      this.currentItem = entree
      this.currentItemType = 'entree'
      this.showSizeModal = true

      this.$nextTick(() => {
        // Focus on the first size option
        const firstSizeButton = this.$refs.entree_sizeButtons?.[0]
        if (firstSizeButton) {
          firstSizeButton.focus()
        }
      })
    },
    async checkPremiumStatus() {
      try {
        // const entreeName = this.getEntreeName(entree) // Get the entree name
        // const encodedEntreeName = encodeURIComponent(entreeName) // Encode the name

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
        // if (isPremium) {
        //   this.premiumEntrees.push(entreeName)
        // } else {
        //   const index = this.premiumEntrees.indexOf(entreeName)
        //   if (index > -1) {
        //     this.premiumEntrees.splice(index, 1)
        //   }
        // }
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
    getEntreeSizeOptions(entree) {
      if (this.isPremium(entree)) {
        const entreeName = this.getEntreeName(entree)
        // Directly map premium size options to return the price from the premiumPrices object
        return Object.keys(this.premiumPrices).map(size => ({
          name: size,
          price: this.premiumPrices[size],
        }))
      }
      return this.sizeOptions.entree // Return default size options for non-premium items
    },
    getSelectedSize(item, type) {
      const selectedList =
        type === 'side' ? this.selectedSides : this.selectedEntrees
      const selectedItem = selectedList.find(selected =>
        selected.name.includes(
          type === 'side' ? this.getSideName(item) : this.getEntreeName(item),
        ),
      )
      return selectedItem ? selectedItem.size : null
    },
    isSelected(item, type) {
      const selectedList =
        type === 'side' ? this.selectedSides : this.selectedEntrees
      return selectedList.some(selected =>
        selected.name.includes(
          this.getSideName(item) || this.getEntreeName(item),
        ),
      )
    },
    async selectSize(size, type) {
      try {
        const selectedList =
          type === 'side' ? this.selectedSides : this.selectedEntrees
        const existingItemIndex = selectedList.findIndex(item =>
          item.name.includes(
            type === 'side'
              ? this.getSideName(this.currentItem)
              : this.getEntreeName(this.currentItem),
          ),
        )

        const foodID = await this.getFoodID(this.currentItem)
        const baseItemID = size.baseItemID

        let price = size.price
        if (type === 'entree' && this.isPremium(this.currentItem)) {
          price = this.premiumPrices[size.name]
        }

        const transactionEntry = [baseItemID, foodID, 0, 0, 0]

        if (existingItemIndex !== -1) {
          selectedList[existingItemIndex] = {
            name: `${type === 'side' ? 'Side' : 'Entree'}: ${type === 'side' ? this.getSideName(this.currentItem) : this.getEntreeName(this.currentItem)} (${size.name})`,
            price,
            size: size.name,
            isPremium: this.isPremium(this.currentItem),
            transactionEntry,
          }
        } else {
          const itemToAdd = {
            name: `${type === 'side' ? 'Side' : 'Entree'}: ${type === 'side' ? this.getSideName(this.currentItem) : this.getEntreeName(this.currentItem)} (${size.name})`,
            price,
            size: size.name,
            isPremium: this.isPremium(this.currentItem),
            transactionEntry,
          }
          selectedList.push(itemToAdd)
        }

        this.showSizeModal = false
        this.currentItem = null

        this.$nextTick(() => {
          const buttonToFocus =
            type === 'side'
              ? this.$refs.sideButtons?.[this.currentItem]
              : this.$refs.entreeButtons?.[this.currentItem]
          if (buttonToFocus) {
            buttonToFocus.focus()
          }
        })

        this.currentItemType = null
      } catch (error) {
        console.error('Error selecting size:', error)
      }
    },
    cancelSizeSelection() {
      this.showSizeModal = false
      this.currentItem = null

      this.$nextTick(() => {
        const buttonToFocus =
          this.currentItemType === 'side'
            ? this.$refs.sideButtons?.[this.currentItem]
            : this.$refs.entreeButtons?.[this.currentItem]
        if (buttonToFocus) {
          buttonToFocus.focus()
        }
      })
    },
    isPremium(entree) {
      return this.premiumEntrees.includes(this.getEntreeName(entree))
    },

    addToCart() {
      if (this.canAddToCart) {
        const itemsToAdd = [...this.selectedSides, ...this.selectedEntrees]

        // Format items to add a label for premium items
        const formattedItems = itemsToAdd.map(item => {
          // Construct the label to include premium status and size
          let label = `${item.name} (${item.size})`

          // If it's a premium item, add "(Premium)" to the label
          if (item.isPremium) {
            label = `${label} (Premium)`
          }

          return {
            ...item,
            label: label, // Add the formatted label to each item
          }
        })

        itemsToAdd.forEach(item => {
          this.$emit('addToCart', item)
          this.$emit('addToTransactionCart', item.transactionEntry)
        })

        // Clear selections after adding to cart
        this.selectedSides = []
        this.selectedEntrees = []
      }
    },
    getSideName(side) {
      return typeof side === 'string'
        ? side
        : side && side.name
          ? side.name
          : 'Unknown Side'
    },
    getSideImage(side) {
      let name = this.getSideName(side)
      if (!name) return null
      const fileName = `${name.toLowerCase().replace(/\s+/g, '')}.png`
      return new URL(`/src/assets/${fileName}`, import.meta.url).href
    },
    getEntreeName(entree) {
      return typeof entree === 'string'
        ? entree
        : entree && entree.name
          ? entree.name
          : 'Unknown Entree'
    },
    getEntreeImage(entree) {
      let name = this.getEntreeName(entree)
      if (!name) return null
      const fileName = `${name.toLowerCase().replace(/\s+/g, '')}.png`
      return new URL(`/src/assets/${fileName}`, import.meta.url).href
    },
    handleImageError(event) {
      console.error('Image failed to load:', event.target.src)
      event.target.style.display = 'none'
    },
  },
  mounted() {
    this.fetchMenuItems() // Fetch menu items when the component mounts
    this.fetchItemPrices()
    this.checkPremiumStatus()
  },
}
</script>

<style scoped>
.alacarte {
  padding: 20px;
}

.alacarte span {
  font-size: em;
}

.grid span {
  font-size: 1em;
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

.size-modal button:focus {
  outline: 2px solid #007bff; /* Blue outline */
  outline-offset: 1px;
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
