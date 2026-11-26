<script setup>
import { ref, computed, onMounted } from 'vue';

const API_BASE = 'https://m01033601-cst3144-after-school.onrender.com';

const lessons = ref([]);
const loading = ref(true);
const errorMessage = ref('');

// Sorting state
const sortBy = ref('subject'); // 'subject' | 'location' | 'price' | 'space'
const sortOrder = ref('asc'); // 'asc' | 'desc'

// View & cart state
const showCart = ref(false);
const cart = ref([]); // items: { lessonId, topic, location, price, qty }

// Checkout state
const customerName = ref('');
const customerPhone = ref('');
const checkoutMessage = ref('');

const sortedLessons = computed(() => {
  const list = [...lessons.value];
  const attr = sortBy.value;
  const dir = sortOrder.value === 'asc' ? 1 : -1;

  return list.sort((a, b) => {
    const valA = a[attr];
    const valB = b[attr];

    if (typeof valA === 'number' && typeof valB === 'number') {
      return (valA - valB) * dir;
    }
    return String(valA ?? '').localeCompare(String(valB ?? '')) * dir;
  });
});

const searchQuery = ref('');
let searchTimer = null;

function fetchLessons(query = '') {
  loading.value = true;
  errorMessage.value = '';

  const trimmed = query.trim();
  const url = trimmed
    ? `${API_BASE}/search?q=${encodeURIComponent(trimmed)}`
    : `${API_BASE}/lessons`;

  fetch(url)
    .then((response) => {
      if (!response.ok) {
        throw new Error(`HTTP error ${response.status}`);
      }
      return response.json();
    })
    .then((data) => {
      lessons.value = data.map((item) => ({
        ...item,
        id: item._id || item.id
      }));
      loading.value = false;
    })
    .catch((err) => {
      errorMessage.value = `Failed to load lessons: ${err.message}`;
      loading.value = false;
    });
}

function onSearchInput() {
  clearTimeout(searchTimer);
  searchTimer = setTimeout(() => {
    fetchLessons(searchQuery.value);
  }, 200);
}

function addToCart(lesson) {
  if (lesson.space <= 0) return;

  lesson.space -= 1;

  const existing = cart.value.find((item) => item.lessonId === lesson.id);
  if (existing) {
    existing.qty += 1;
  } else {
    cart.value.push({
      lessonId: lesson.id,
      topic: lesson.topic,
      location: lesson.location,
      price: lesson.price,
      qty: 1
    });
  }
}

function removeFromCart(index) {
  const item = cart.value[index];
  if (!item) return;

  const lesson = lessons.value.find((l) => l.id === item.lessonId);
  if (lesson) {
    lesson.space += 1;
  }

  if (item.qty > 1) {
    item.qty -= 1;
  } else {
    cart.value.splice(index, 1);
  }

  if (cart.value.length === 0) {
    showCart.value = false;
  }
}

function toggleCart() {
  if (cart.value.length === 0) return;
  checkoutMessage.value = '';
  showCart.value = !showCart.value;
}

const isNameValid = computed(() => /^[A-Za-z\s]+$/.test(customerName.value.trim()));
const isPhoneValid = computed(() => /^[0-9]+$/.test(customerPhone.value.trim()));
const canCheckout = computed(
  () => cart.value.length > 0 && isNameValid.value && isPhoneValid.value
);

function submitOrder() {
  if (!canCheckout.value) return;

  const totalSpaces = cart.value.reduce((sum, item) => sum + item.qty, 0);
  const lessonIDs = cart.value.map((item) => item.lessonId);

  const orderPayload = {
    name: customerName.value.trim(),
    phone: customerPhone.value.trim(),
    lessonIDs,
    space: totalSpaces
  };

  fetch(`${API_BASE}/orders`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(orderPayload)
  })
    .then((response) => {
      if (!response.ok) {
        throw new Error(`Order failed (${response.status})`);
      }
      return response.json();
    })
    .then(() => {
      const spaceUpdates = cart.value.map((item) => {
        const lesson = lessons.value.find((l) => l.id === item.lessonId);
        const updatedSpace = lesson ? lesson.space : 0;

        return fetch(`${API_BASE}/lessons/${item.lessonId}`, {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ space: updatedSpace })
        });
      });

      return Promise.all(spaceUpdates);
    })
    .then(() => {
      checkoutMessage.value = `Order confirmed for ${customerName.value.trim()}!`;
      cart.value = [];
      customerName.value = '';
      customerPhone.value = '';
      showCart.value = false;
      fetchLessons();
    })
    .catch((err) => {
      checkoutMessage.value = `Error submitting order: ${err.message}`;
    });
}
</script>

<template>
  <main class="container">
    <header class="app-hero">
      <h1>After-School Lessons</h1>
      <button
        type="button"
        :disabled="cart.length === 0"
        v-on:click="toggleCart"
      >
        {{ showCart ? 'Back to Lessons' : `Shopping Cart (${cart.reduce((s, i) => s + i.qty, 0)})` }}
      </button>
    </header>

    <p v-if="checkoutMessage" class="confirmation">{{ checkoutMessage }}</p>
    <p v-if="errorMessage" class="error">{{ errorMessage }}</p>
    <p v-if="loading">Loading lessons...</p>

    <!-- Lessons Page -->
    <section v-else-if="!showCart">
      <div class="controls-bar">
        <label>
          Sort by:
          <select v-model="sortBy">
            <option value="subject">Subject</option>
            <option value="location">Location</option>
            <option value="price">Price</option>
            <option value="space">Spaces</option>
          </select>
        </label>

        <label>
          Order:
          <select v-model="sortOrder">
            <option value="asc">Ascending</option>
            <option value="desc">Descending</option>
          </select>
        </label>

        <label>
          Search:
          <input
            type="text"
            v-model="searchQuery"
            v-on:input="onSearchInput"
            placeholder="Search topic, location, price, spaces..."
          />
        </label>
      </div>

      <div class="lesson-grid">
        <article v-for="lesson in sortedLessons" :key="lesson.id" class="lesson-card">
          <img
            v-if="lesson.image"
            :src="lesson.image.startsWith('http') ? lesson.image : `${API_BASE}/images/${lesson.image}`"
            :alt="lesson.topic || lesson.subject"
            class="lesson-img"
          />
          <h2>{{ lesson.topic || lesson.subject }}</h2>
          <p><strong>Location:</strong> {{ lesson.location }}</p>
          <p><strong>Price:</strong> £{{ lesson.price }}</p>
          <p><strong>Spaces:</strong> {{ lesson.space }}</p>
          <button
            type="button"
            :disabled="lesson.space <= 0"
            v-on:click="addToCart(lesson)"
          >
            Add to Cart
          </button>
        </article>
      </div>
    </section>

    <!-- Shopping Cart & Checkout Page -->
    <section v-else class="cart-panel">
      <h2>Shopping Cart</h2>
      <ul>
        <li v-for="(item, index) in cart" :key="item.lessonId" class="cart-item">
          <span>{{ item.topic }} ({{ item.location }}) — £{{ item.price }} × {{ item.qty }}</span>
          <button type="button" v-on:click="removeFromCart(index)">Remove</button>
        </li>
      </ul>

      <div class="checkout-panel">
        <h3>Checkout</h3>
        <label>
          Name (letters only):
          <input type="text" v-model="customerName" placeholder="e.g. Sarah Jenkins" />
        </label>
        <p v-if="customerName && !isNameValid" class="hint">Name must contain letters only.</p>

        <label>
          Phone (numbers only):
          <input type="text" v-model="customerPhone" placeholder="e.g. 07123456789" />
        </label>
        <p v-if="customerPhone && !isPhoneValid" class="hint">Phone must contain numbers only.</p>

        <button
          type="button"
          :disabled="!canCheckout"
          v-on:click="submitOrder"
        >
          Checkout
        </button>
      </div>
    </section>
  </main>
</template>