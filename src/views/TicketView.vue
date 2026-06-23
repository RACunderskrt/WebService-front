<template>
  	<div class="app">
    	<!-- Header -->
    	<header class="header">
      		<div class="header-inner">
        		<div class="header-left">
					<div class="brand">
						<span class="brand-icon">✈</span>
						<span class="brand-name">AirBoard</span>
					</div>
					<p class="header-sub">Available Flights & Seat Inventory</p>
				</div>

        		<!-- Auth -->
				<div class="header-auth">
					<!-- Logged in -->
					<div v-if="user" class="user-info">
						<div class="user-avatar">{{ userInitial }}</div>
						<div class="user-details">
						<span class="user-name">{{ user.name }}</span>
						<span class="user-email">{{ user.email }}</span>
						</div>
					</div>
					<!-- Logged out -->
					<div v-else class="login-buttons">
						<button class="btn-login btn-login--google" @click="loginWithGoogle">
							<svg class="google-icon" viewBox="0 0 24 24" xmlns="http://www.w3.org/2000/svg">
								<path d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92c-.26 1.37-1.04 2.53-2.21 3.31v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.09z" fill="#4285F4"/>
								<path d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" fill="#34A853"/>
								<path d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18C1.43 8.55 1 10.22 1 12s.43 3.45 1.18 4.93l3.66-2.84z" fill="#FBBC05"/>
								<path d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" fill="#EA4335"/>
							</svg>
							Log in with Google
						</button>

						<button class="btn-login btn-login--keycloak" @click="loginWithKeycloak">
							<!-- Keycloak logo mark -->
							<svg class="keycloak-icon" viewBox="0 0 64 64" xmlns="http://www.w3.org/2000/svg">
								<path d="M32 4L6 18v28l26 14 26-14V18L32 4z" fill="#4D9BF5"/>
								<path d="M32 12l-18 10v20l18 10 18-10V22L32 12z" fill="#fff" opacity="0.15"/>
								<path d="M24 26h6v4l6-4h4v12h-4v-7l-6 4v3h-6V26z" fill="#fff"/>
							</svg>
							Log in with Keycloak
						</button>
					</div>
				</div>
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
					:company-name="getCompanyName(flight.companyId)"
					@book-seat="bookSeat"
					@delete-seat="deleteSeat"
				/>
			</div>
		</main>
	</div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import TicketComponent from '@/components/TicketComponent.vue'
import Keycloak from 'keycloak-js'

const keycloak = new Keycloak({
	url:      'http://localhost:8080',
	realm:    'master',
	clientId: 'web-service-api',
})

const user = ref(null)
const compagnies = ref([])
const vols = ref([])
const sieges = ref([])
const search = ref('')
const filterAvailability = ref('all')

const userInitial = computed(() =>
  	user.value?.name?.charAt(0).toUpperCase() ?? '?'
)

onMounted(async () => {
	try {
		const authenticated = await keycloak.init({
			onLoad: 'check-sso',
			silentCheckSsoRedirectUri: window.location.origin + '/silent-check-sso.html',
			pkceMethod: 'S256',
		})

		if (authenticated) {
			user.value = {
				name:  keycloak.tokenParsed?.name  ?? keycloak.tokenParsed?.preferred_username,
				email: keycloak.tokenParsed?.email,
			}
			await getData();
		}
	} catch {
		// Keycloak unreachable — continue as guest
	}
})

async function getData(){
	await getCompanies();
	await getFlights();
	await getSeatsAPI();
}

async function getCompanies(){
	const res = await fetch('http://localhost:3000/company', {
		headers: {
			Authorization: `Bearer ${keycloak.token}`
		}
	})
	compagnies.value = await res.json();
	console.log("compagnies.value", compagnies.value)
}

async function getFlights(){
	const res = await fetch('http://localhost:3000/flight', {
		headers: {
			Authorization: `Bearer ${keycloak.token}`
		}
	})
	vols.value = await res.json();
	console.log("vols.value", vols.value)
}

async function deleteSeat(seat) {
	console.log("delete")
	try {
		const res = await fetch(`http://localhost:3000/seat/${seat.id}`, {
			method: 'DELETE',
			headers: {
				Authorization: `Bearer ${keycloak.token}`,
			},
		})
		if (!res.ok) throw new Error(`Server error: ${res.status}`)
		await getSeatsAPI()
	} catch (err) {
		console.error('Failed to delete seat:', err)
	}
}

async function bookSeat(seat) {
	console.log("book")
	try {
		const res = await fetch('http://localhost:3000/seat', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
				Authorization: `Bearer ${keycloak.token}`,
			},
			body: JSON.stringify({
				id:              seat.id,
				flightId:        seat.flightId,
				label:           seat.label,
				clientName:      user.value.name.split(" ")[1],
				clientFirstname: user.value.name.split(" ")[0],
			}),
		})
		if (!res.ok) throw new Error(`Server error: ${res.status}`)
		await getSeatsAPI()
	} catch (err) {
		console.error('Failed to book seat:', err)
	}
}

async function getSeatsAPI(){
	const res = await fetch('http://localhost:3000/seat', {
		headers: {
			Authorization: `Bearer ${keycloak.token}`
		}
	})
	sieges.value = await res.json();
	console.log("sieges.value", sieges.value)
}


const loginWithGoogle = () => {
	const width  = 500
	const height = 600
	const left   = window.screenX + (window.outerWidth  - width)  / 2
	const top    = window.screenY + (window.outerHeight - height) / 2

	const popup = window.open(
		'http://localhost:3000/oauthgoogle',
		'google-oauth',
		`width=${width},height=${height},left=${left},top=${top},resizable=yes,scrollbars=yes`
	)

	const onMessage = (event) => {
		if (event.origin !== 'http://localhost:3000') return
		if (!event.data?.email) return
		user.value = event.data
		window.removeEventListener('message', onMessage)
		if (popup && !popup.closed) popup.close()
	}

	window.addEventListener('message', onMessage)

	const pollClosed = setInterval(() => {
		if (popup?.closed) {
			window.removeEventListener('message', onMessage)
			clearInterval(pollClosed)
		}
	}, 500)
}

const loginWithKeycloak = () => {
  	keycloak.login()
}

const getSeats = (flight) =>
	sieges.value
		.filter(s => s.flightId === flight.id)
		.map(s => ({ ...s, occupied: !!s.clientName }))

const getCompanyName = (id) => {
  	const c = compagnies.value.find(c => c.id === id)
  	return c ? c.name : '—'
}

const filteredFlights = computed(() =>
  	vols.value.filter(flight => {
		const q = search.value.toLowerCase()
		const matchSearch =
			!q ||
			flight.departureAirport.toLowerCase().includes(q) ||
			flight.arrivalAirport.toLowerCase().includes(q) ||
			getCompanyName(flight.companyId).toLowerCase().includes(q)

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
@import url('https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;500;600;700&family=JetBrains+Mono:wght@400;500&display=swap');

:root {
  --sky:    #0A1628;
  --mid:    #102040;
  --card:   #13263E;
  --line:   #1E3A5A;
  --accent: #4DB8FF;
  --go:     #22D98C;
  --warn:   #F59E0B;
  --full:   #EF4444;
  --text:   #E8F4FF;
  --muted:  #7AAAC8;
  --mono:   'JetBrains Mono', monospace;
  --sans:   'Space Grotesk', sans-serif;
}

*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

.app {
  min-height: 100vh;
  background: var(--sky);
  font-family: var(--sans);
  color: var(--text);
}

/* Header */
.header {
  background: linear-gradient(135deg, var(--mid) 0%, #0D1E38 100%);
  border-bottom: 1px solid var(--line);
  padding: 28px 0 24px;
}
.header-inner { max-width: 1100px; margin: 0 auto; padding: 0 24px; display: flex; align-items: center; justify-content: space-between; gap: 16px; }
.header-left { display: flex; flex-direction: column; }
.brand { display: flex; align-items: center; gap: 10px; margin-bottom: 4px; }
.brand-icon { font-size: 22px; color: var(--accent); }
.brand-name { font-size: 22px; font-weight: 700; letter-spacing: -0.5px; color: var(--text); }
.header-sub { font-size: 13px; color: var(--muted); letter-spacing: 0.05em; text-transform: uppercase; }

/* Filters */
.filters-bar {
  background: var(--mid);
  border-bottom: 1px solid var(--line);
  padding: 14px 0;
  position: sticky;
  top: 0;
  z-index: 10;
}
.filters-inner { max-width: 1100px; margin: 0 auto; padding: 0 24px; display: flex; gap: 12px; }
.search-input,
.filter-select {
  background: var(--sky);
  border: 1px solid var(--line);
  color: var(--text);
  font-family: var(--sans);
  font-size: 14px;
  padding: 8px 14px;
  border-radius: 6px;
  outline: none;
  transition: border-color 0.2s;
}
.search-input { flex: 1; }
.search-input::placeholder { color: var(--muted); }
.search-input:focus,
.filter-select:focus { border-color: var(--accent); }
.filter-select option { background: var(--sky); }

/* Main */
.main { max-width: 1100px; margin: 0 auto; padding: 32px 24px 64px; }
.empty-state { text-align: center; padding: 80px 0; color: var(--muted); }
.empty-icon  { font-size: 40px; display: block; margin-bottom: 12px; }

/* Grid */
.flights-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
}

/* Auth */
.header-auth { flex-shrink: 0; }

.login-buttons {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  justify-content: flex-end;
}

.btn-login {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 9px 16px;
  font-family: var(--sans);
  font-size: 14px;
  font-weight: 600;
  border-radius: 8px;
  border: none;
  cursor: pointer;
  transition: box-shadow 0.15s, opacity 0.15s;
  white-space: nowrap;
}
.btn-login:hover { box-shadow: 0 2px 8px rgba(0,0,0,0.3); opacity: 0.92; }

/* Google */
.btn-login--google {
  background: #fff;
  color: #3c3c3c;
  border: 1px solid #dadce0;
}
.google-icon { width: 18px; height: 18px; flex-shrink: 0; }

/* Keycloak */
.btn-login--keycloak {
  background: #4D9BF5;
  color: #fff;
  border: 1px solid #3a85d8;
}
.keycloak-icon { width: 20px; height: 20px; flex-shrink: 0; }

.user-info {
  display: flex;
  align-items: center;
  gap: 12px;
}
.user-avatar {
  width: 38px;
  height: 38px;
  border-radius: 50%;
  background: var(--accent);
  color: var(--sky);
  font-size: 16px;
  font-weight: 700;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.user-details { display: flex; flex-direction: column; }
.user-name  { font-size: 14px; font-weight: 600; color: var(--text); }
.user-email { font-size: 12px; color: var(--muted); }
</style>