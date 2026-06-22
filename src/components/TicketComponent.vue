<template>
  <div
    class="flight-card"
    :class="{ 'flight-card--full': availableSeats === 0 }"
  >
    <!-- Route -->
    <div class="card-route">
      <div class="route-point">
        <span class="route-code">{{ flight.lieu_depart }}</span>
        <span class="route-label">Departure</span>
      </div>
      <div class="route-line">
        <span class="route-plane">✈</span>
      </div>
      <div class="route-point route-point--right">
        <span class="route-code">{{ flight.lieu_arrivee }}</span>
        <span class="route-label">Arrival</span>
      </div>
    </div>

    <!-- Dates -->
    <div class="card-dates">
      <div class="date-block">
        <span class="date-value">{{ formatDate(flight.date_depart) }}</span>
        <span class="date-time">{{ formatTime(flight.date_depart) }}</span>
      </div>
      <div class="duration-block">
        <span class="duration">{{ duration }}</span>
      </div>
      <div class="date-block date-block--right">
        <span class="date-value">{{ formatDate(flight.date_arrivee) }}</span>
        <span class="date-time">{{ formatTime(flight.date_arrivee) }}</span>
      </div>
    </div>

    <!-- Company -->
    <div class="card-company">
      <span class="company-label">Operated by</span>
      <span class="company-name">{{ companyName }}</span>
    </div>

    <!-- Seats -->
    <div class="card-seats">
      <div class="seats-summary">
        <span class="seats-badge" :class="seatBadgeClass">
          {{ availableSeats }} / {{ seats.length }} seats free
        </span>
      </div>
      <div class="seats-grid">
        <div
          v-for="seat in seats"
          :key="seat.id"
          class="seat"
          :class="seat.occupied ? 'seat--taken' : 'seat--free'"
          :title="seat.occupied ? `${seat.firstname_client} ${seat.name_client}` : 'Available'"
        >
          {{ seat.seat_label }}
        </div>
      </div>
    </div>

    <!-- Action -->
    <div class="card-action">
      <button class="btn-book" :disabled="availableSeats === 0">
        {{ availableSeats === 0 ? 'Fully Booked' : 'Book a Seat' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

// ── Props ────────────────────────────────────────────────────────────────────
const props = defineProps({
  flight: {
    type: Object,
    required: true,
  },
  // Array of siege objects already filtered for this flight
  seats: {
    type: Array,
    required: true,
  },
  companyName: {
    type: String,
    default: '—',
  },
})

// ── Computed ─────────────────────────────────────────────────────────────────
const availableSeats = computed(() => props.seats.filter(s => !s.occupied).length)

const duration = computed(() => {
  const diff = (new Date(props.flight.date_arrivee) - new Date(props.flight.date_depart)) / 60000
  const h = Math.floor(diff / 60)
  const m = diff % 60
  return `${h}h${String(m).padStart(2, '0')}`
})

const seatBadgeClass = computed(() => {
  const ratio = availableSeats.value / props.seats.length
  if (ratio === 0)   return 'badge--full'
  if (ratio < 0.35)  return 'badge--low'
  return 'badge--ok'
})

// ── Helpers ──────────────────────────────────────────────────────────────────
const formatDate = (iso) =>
  new Date(iso).toLocaleDateString('fr-FR', { day: '2-digit', month: 'short', year: 'numeric' })

const formatTime = (iso) =>
  new Date(iso).toLocaleTimeString('fr-FR', { hour: '2-digit', minute: '2-digit' })
</script>

<style scoped>
@import "src/style/style.css"
</style>