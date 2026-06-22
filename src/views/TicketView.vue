<template>
  <div class="app">
    <!-- Header -->
    <header class="header">
      <div class="header-inner">
        <div class="brand">
          <span class="brand-icon">✈</span>
          <span class="brand-name">AirBoard</span>
        </div>
        <p class="header-sub">Available Flights & Seat Inventory</p>
      </div>
    </header>

    <!-- Filters -->
    <div class="filters-bar">
      <div class="filters-inner">
        <input
          v-model="search"
          class="search-input"
          placeholder="Search destination or origin…"
        />
        <select v-model="filterAvailability" class="filter-select">
          <option value="all">All flights</option>
          <option value="available">Has available seats</option>
          <option value="full">Fully booked</option>
        </select>
      </div>
    </div>

    <!-- Flight list -->
    <main class="main">
      <div v-if="filteredFlights.length === 0" class="empty-state">
        <span class="empty-icon">🔍</span>
        <p>No flights match your search. Try adjusting the filters.</p>
      </div>

      <div class="flights-grid">
        <TicketComponent
          v-for="flight in filteredFlights"
          :key="flight.id"
          :flight="flight"
          :seats="getSeats(flight)"
          :company-name="getCompanyName(flight.id_compagnie)"
        />
      </div>
    </main>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import TicketComponent from '@/components/TicketComponent.vue'

// ── Reactive state ───────────────────────────────────────────────────────────
const search             = ref('')
const filterAvailability = ref('all')

// ── Mock data (replace with API calls as needed) ─────────────────────────────
const compagnies = ref([
  { id: 1, name: 'Air Méditerranée', postal_code: '13001', ville: 'Marseille', email: 'contact@airmed.fr', phone_number: '+33 4 91 00 00 01' },
  { id: 2, name: 'SkyLine Express',  postal_code: '69002', ville: 'Lyon',      email: 'info@skyline.fr',  phone_number: '+33 4 72 00 00 02' },
  { id: 3, name: 'OcciJet',          postal_code: '31000', ville: 'Toulouse',  email: 'ops@occijet.fr',   phone_number: '+33 5 61 00 00 03' },
])

const vols = ref([
  { id: 101, id_compagnie: 1, lieu_depart: 'CDG', date_depart: '2026-07-10T08:30:00', lieu_arrivee: 'NCE', date_arrivee: '2026-07-10T10:05:00' },
  { id: 102, id_compagnie: 2, lieu_depart: 'LYS', date_depart: '2026-07-11T14:00:00', lieu_arrivee: 'MRS', date_arrivee: '2026-07-11T15:20:00' },
  { id: 103, id_compagnie: 3, lieu_depart: 'TLS', date_depart: '2026-07-12T06:45:00', lieu_arrivee: 'ORY', date_arrivee: '2026-07-12T08:10:00' },
  { id: 104, id_compagnie: 1, lieu_depart: 'MRS', date_depart: '2026-07-13T17:55:00', lieu_arrivee: 'CDG', date_arrivee: '2026-07-13T19:30:00' },
])

// name_client / firstname_client === null → seat is free
const sieges = ref([
  // Vol 101 — 3 booked out of 6
  { id: 1,  id_vol: 101, seat_label: '1A', name_client: 'Martin',  firstname_client: 'Paul'   },
  { id: 2,  id_vol: 101, seat_label: '1B', name_client: null,      firstname_client: null      },
  { id: 3,  id_vol: 101, seat_label: '2A', name_client: 'Dubois',  firstname_client: 'Léa'    },
  { id: 4,  id_vol: 101, seat_label: '2B', name_client: null,      firstname_client: null      },
  { id: 5,  id_vol: 101, seat_label: '3A', name_client: 'Bernard', firstname_client: 'Marc'   },
  { id: 6,  id_vol: 101, seat_label: '3B', name_client: null,      firstname_client: null      },
  // Vol 102 — fully booked
  { id: 7,  id_vol: 102, seat_label: '1A', name_client: 'Petit',   firstname_client: 'Alice'  },
  { id: 8,  id_vol: 102, seat_label: '1B', name_client: 'Robert',  firstname_client: 'Jean'   },
  { id: 9,  id_vol: 102, seat_label: '2A', name_client: 'Moreau',  firstname_client: 'Sophie' },
  { id: 10, id_vol: 102, seat_label: '2B', name_client: 'Simon',   firstname_client: 'Hugo'   },
  // Vol 103 — mostly free
  { id: 11, id_vol: 103, seat_label: '1A', name_client: 'Laurent', firstname_client: 'Eva'    },
  { id: 12, id_vol: 103, seat_label: '1B', name_client: null,      firstname_client: null      },
  { id: 13, id_vol: 103, seat_label: '2A', name_client: null,      firstname_client: null      },
  { id: 14, id_vol: 103, seat_label: '2B', name_client: null,      firstname_client: null      },
  { id: 15, id_vol: 103, seat_label: '3A', name_client: null,      firstname_client: null      },
  { id: 16, id_vol: 103, seat_label: '3B', name_client: null,      firstname_client: null      },
  // Vol 104 — half booked
  { id: 17, id_vol: 104, seat_label: '1A', name_client: 'Garnier', firstname_client: 'Nora'   },
  { id: 18, id_vol: 104, seat_label: '1B', name_client: null,      firstname_client: null      },
  { id: 19, id_vol: 104, seat_label: '2A', name_client: 'Faure',   firstname_client: 'Tom'    },
  { id: 20, id_vol: 104, seat_label: '2B', name_client: null,      firstname_client: null      },
])

// ── Helpers ──────────────────────────────────────────────────────────────────
const getSeats = (flight) =>
  sieges.value
    .filter(s => s.id_vol === flight.id)
    .map(s => ({ ...s, occupied: !!s.name_client }))

const getCompanyName = (id) => {
  const c = compagnies.value.find(c => c.id === id)
  return c ? c.name : '—'
}

// ── Computed ──────────────────────────────────────────────────────────────────
const filteredFlights = computed(() =>
  vols.value.filter(flight => {
    const q = search.value.toLowerCase()
    const matchSearch =
      !q ||
      flight.lieu_depart.toLowerCase().includes(q) ||
      flight.lieu_arrivee.toLowerCase().includes(q) ||
      getCompanyName(flight.id_compagnie).toLowerCase().includes(q)

    const freeSeats = getSeats(flight).filter(s => !s.occupied).length
    const matchAvail =
      filterAvailability.value === 'all' ||
      (filterAvailability.value === 'available' && freeSeats > 0) ||
      (filterAvailability.value === 'full'      && freeSeats === 0)

    return matchSearch && matchAvail
  })
)
</script>

<style scoped>
@import "src/style/style.css"
</style>