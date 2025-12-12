<script setup>
import { reactive, ref, computed, onMounted, watch } from "vue";

/* -----------------------------
   State
------------------------------ */
const crime_url = ref("");
const dialog_err = ref(false);
const dialogRef = ref(null);

const map = reactive({
  leaflet: null,
  center: { lat: 44.955139, lng: -93.102222, address: "" },
  zoom: 12,

  // St. Paul bounding box (NW/SE)
  bounds: {
    nw: { lat: 45.008206, lng: -93.217977 },
    se: { lat: 44.883658, lng: -92.993787 },
  },

  // NOTE: we’ll treat these as neighborhood IDs 1..17 in this order
  neighborhood_markers: [
    { id: 1, location: [44.942068, -93.020521], marker: null },
    { id: 2, location: [44.977413, -93.025156], marker: null },
    { id: 3, location: [44.931244, -93.079578], marker: null },
    { id: 4, location: [44.956192, -93.060189], marker: null },
    { id: 5, location: [44.978883, -93.068163], marker: null },
    { id: 6, location: [44.975766, -93.113887], marker: null },
    { id: 7, location: [44.959639, -93.121271], marker: null },
    { id: 8, location: [44.9477, -93.128505], marker: null },
    { id: 9, location: [44.930276, -93.119911], marker: null },
    { id: 10, location: [44.982752, -93.14791], marker: null },
    { id: 11, location: [44.963631, -93.167548], marker: null },
    { id: 12, location: [44.973971, -93.197965], marker: null },
    { id: 13, location: [44.949043, -93.178261], marker: null },
    { id: 14, location: [44.934848, -93.176736], marker: null },
    { id: 15, location: [44.913106, -93.170779], marker: null },
    { id: 16, location: [44.937705, -93.136997], marker: null },
    { id: 17, location: [44.949203, -93.093739], marker: null },
  ],
});

const ui = reactive({
  // location box (address OR "lat,lng")
  locationText: "",
  locationErr: "",

  // crimes
  isReady: false,
  loading: false,
  apiErr: "",
});

const lookups = reactive({
  // code -> incident_type
  codeToType: new Map(),
  // neighborhood_number -> neighborhood_name
  nhoodToName: new Map(),
});

const crimes = ref([]); // raw incidents from API
const visibleNeighborhoodIds = ref(new Set()); // ids currently visible in map view

const upload = reactive({
  case_number: "",
  date: "",
  time: "",
  code: "",
  incident: "",
  police_grid: "",
  neighborhood_number: "",
  block: "",
  err: "",
  ok: "",
  submitting: false,
});

/* -----------------------------
   Helpers
------------------------------ */
function clampLatLng(lat, lng) {
  const minLat = map.bounds.se.lat;
  const maxLat = map.bounds.nw.lat;
  const minLng = map.bounds.nw.lng;
  const maxLng = map.bounds.se.lng;

  const clampedLat = Math.min(maxLat, Math.max(minLat, lat));
  const clampedLng = Math.min(maxLng, Math.max(minLng, lng));
  return { lat: clampedLat, lng: clampedLng };
}

function parseLatLng(text) {
  // accepts: "44.95,-93.10" or "44.95 -93.10"
  const cleaned = text.trim().replace(/\s+/g, " ");
  const parts = cleaned.includes(",") ? cleaned.split(",") : cleaned.split(" ");
  if (parts.length !== 2) return null;

  const lat = Number(parts[0]);
  const lng = Number(parts[1]);
  if (!Number.isFinite(lat) || !Number.isFinite(lng)) return null;
  return { lat, lng };
}

function splitDateTime(date_time) {
  // date_time comes back like "YYYY-MM-DD HH:MM:SS"
  const dt = String(date_time || "");
  return {
    date: dt.slice(0, 10),
    time: dt.slice(11, 19),
  };
}

/* -----------------------------
   Nominatim
------------------------------ */
async function nominatimSearch(q) {
  const url =
    "https://nominatim.openstreetmap.org/search?" +
    new URLSearchParams({
      q,
      format: "json",
      limit: "1",
    });
  const r = await fetch(url, { headers: { Accept: "application/json" } });
  if (!r.ok) throw new Error("Nominatim search failed");
  const data = await r.json();
  if (!data || data.length === 0) return null;
  return {
    lat: Number(data[0].lat),
    lng: Number(data[0].lon),
    display: data[0].display_name || "",
  };
}

async function nominatimReverse(lat, lng) {
  const url =
    "https://nominatim.openstreetmap.org/reverse?" +
    new URLSearchParams({
      lat: String(lat),
      lon: String(lng),
      format: "json",
      zoom: "18",
    });
  const r = await fetch(url, { headers: { Accept: "application/json" } });
  if (!r.ok) throw new Error("Nominatim reverse failed");
  const data = await r.json();
  return data?.display_name || `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
}

/* -----------------------------
   Crime API calls
------------------------------ */
async function fetchCodes() {
  const r = await fetch(`${crime_url.value}/codes`);
  if (!r.ok) throw new Error("Failed to load /codes");
  const data = await r.json();
  lookups.codeToType.clear();
  data.forEach((row) => {
    // your API returns {code, type}
    lookups.codeToType.set(Number(row.code), String(row.type));
  });
}

async function fetchNeighborhoods() {
  const r = await fetch(`${crime_url.value}/neighborhoods`);
  if (!r.ok) throw new Error("Failed to load /neighborhoods");
  const data = await r.json();
  lookups.nhoodToName.clear();
  data.forEach((row) => {
    // your API returns {id, name}
    lookups.nhoodToName.set(Number(row.id), String(row.name));
  });
}

async function fetchIncidents(limit = 1000) {
  const r = await fetch(`${crime_url.value}/incidents?limit=${limit}`);
  if (!r.ok) throw new Error("Failed to load /incidents");
  const data = await r.json();

  // your /incidents returns rows with date_time (not split)
  crimes.value = (data || []).map((row) => {
    const { date, time } = splitDateTime(row.date_time);
    return {
      case_number: row.case_number,
      date,
      time,
      code: Number(row.code),
      incident: row.incident,
      police_grid: row.police_grid,
      neighborhood_number: Number(row.neighborhood_number),
      block: row.block,
    };
  });
}

/* -----------------------------
   Visibility filtering
------------------------------ */
function recomputeVisibleNeighborhoods() {
  if (!map.leaflet) return;

  const b = map.leaflet.getBounds(); // Leaflet LatLngBounds
  const visible = new Set();

  for (const nm of map.neighborhood_markers) {
    const lat = nm.location[0];
    const lng = nm.location[1];
    if (b.contains([lat, lng])) visible.add(nm.id);
  }

  visibleNeighborhoodIds.value = visible;
}

const crimesInView = computed(() => {
  // requirement: only show crimes that occurred in neighborhoods visible on the map
  const visible = visibleNeighborhoodIds.value;
  if (!visible || visible.size === 0) return [];
  return crimes.value.filter((c) => visible.has(c.neighborhood_number));
});

const crimeCountsByNeighborhood = computed(() => {
  const counts = new Map();
  for (const c of crimesInView.value) {
    const id = c.neighborhood_number;
    counts.set(id, (counts.get(id) || 0) + 1);
  }
  return counts;
});

function updateNeighborhoodPopups() {
  const counts = crimeCountsByNeighborhood.value;
  for (const nm of map.neighborhood_markers) {
    if (!nm.marker) continue;
    const nName = lookups.nhoodToName.get(nm.id) || `Neighborhood ${nm.id}`;
    const count = counts.get(nm.id) || 0;
    nm.marker.bindPopup(`<b>${nName}</b><br/>Crimes in view: ${count}`);
  }
}

/* -----------------------------
   Map controls
------------------------------ */
async function goToLocation() {
  ui.locationErr = "";
  const text = ui.locationText.trim();
  if (!text) {
    ui.locationErr = "Enter an address or lat,lng.";
    return;
  }

  try {
    const parsed = parseLatLng(text);
    let lat, lng;

    if (parsed) {
      ({ lat, lng } = parsed);
    } else {
      const hit = await nominatimSearch(text);
      if (!hit) {
        ui.locationErr = "No results found for that location.";
        return;
      }
      lat = hit.lat;
      lng = hit.lng;
    }

    const clamped = clampLatLng(lat, lng);
    map.leaflet.setView([clamped.lat, clamped.lng], Math.max(map.leaflet.getZoom(), 12));

    // update address box to something nice after moving
    // (moveend handler also does this; this is just “fast feedback”)
    ui.locationText = `${clamped.lat.toFixed(6)}, ${clamped.lng.toFixed(6)}`;
  } catch (e) {
    ui.locationErr = "Could not geocode that location.";
  }
}

async function handleMoveEnd() {
  if (!map.leaflet) return;

  // update center
  const c = map.leaflet.getCenter();
  const clamped = clampLatLng(c.lat, c.lng);
  if (clamped.lat !== c.lat || clamped.lng !== c.lng) {
    // if user managed to drag near edge, snap inside bounds
    map.leaflet.panTo([clamped.lat, clamped.lng], { animate: false });
  }

  // recompute what neighborhoods are visible
  recomputeVisibleNeighborhoods();
  updateNeighborhoodPopups();

  // update input box based on reverse geocode (recommended after pan/zoom ends)
  try {
    const addr = await nominatimReverse(clamped.lat, clamped.lng);
    ui.locationText = addr;
  } catch {
    ui.locationText = `${clamped.lat.toFixed(6)}, ${clamped.lng.toFixed(6)}`;
  }
}

/* -----------------------------
   Markers
------------------------------ */
function initializeNeighborhoodMarkers() {
  for (const nm of map.neighborhood_markers) {
    if (nm.marker) continue;
    nm.marker = L.marker(nm.location).addTo(map.leaflet);
  }
  updateNeighborhoodPopups();
}

/* -----------------------------
   Startup: API initialization
------------------------------ */
async function initializeCrimes() {
  ui.apiErr = "";
  ui.loading = true;

  try {
    await fetchCodes();
    await fetchNeighborhoods();
    await fetchIncidents(1000);

    initializeNeighborhoodMarkers();
    recomputeVisibleNeighborhoods();
    updateNeighborhoodPopups();

    ui.isReady = true;
  } catch (e) {
    ui.apiErr = "Could not load data from your Crime API. Check the URL and that the server is running.";
    ui.isReady = false;
  } finally {
    ui.loading = false;
  }
}

function closeDialog() {
  const url = crime_url.value.trim();
  if (!url) {
    dialog_err.value = true;
    return;
  }
  dialog_err.value = false;
  dialogRef.value?.close();
  initializeCrimes();
}

/* -----------------------------
   Upload new incident
------------------------------ */
async function submitIncident() {
  upload.err = "";
  upload.ok = "";

  // required fields
  const required = [
    ["case_number", upload.case_number],
    ["date", upload.date],
    ["time", upload.time],
    ["code", upload.code],
    ["incident", upload.incident],
    ["police_grid", upload.police_grid],
    ["neighborhood_number", upload.neighborhood_number],
    ["block", upload.block],
  ];

  const missing = required.filter(([, v]) => String(v).trim() === "");
  if (missing.length) {
    upload.err = "Please fill out all fields before submitting.";
    return;
  }

  upload.submitting = true;
  try {
    const body = {
      case_number: String(upload.case_number).trim(),
      date: String(upload.date).trim(),
      time: String(upload.time).trim(),
      code: Number(upload.code),
      incident: String(upload.incident).trim(),
      police_grid: Number(upload.police_grid),
      neighborhood_number: Number(upload.neighborhood_number),
      block: String(upload.block).trim(),
    };

    const r = await fetch(`${crime_url.value}/new-incident`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });

    const payload = await r.json().catch(() => ({}));

    if (!r.ok) {
      // your API uses 400 for missing, 500 for duplicate
      upload.err = payload?.error || "Upload failed.";
      return;
    }

    upload.ok = `Uploaded incident (${body.case_number}). Refreshing…`;

    // refresh list (still “most recent first”)
    await fetchIncidents(1000);
    recomputeVisibleNeighborhoods();
    updateNeighborhoodPopups();
  } catch (e) {
    upload.err = "Upload failed (network/server error).";
  } finally {
    upload.submitting = false;
  }
}

/* -----------------------------
   Leaflet init
------------------------------ */
onMounted(() => {
  // Create Leaflet map
  map.leaflet = L.map("leafletmap").setView([map.center.lat, map.center.lng], map.zoom);

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    minZoom: 11,
    maxZoom: 18,
  }).addTo(map.leaflet);

  // prevent panning outside St Paul
  map.leaflet.setMaxBounds([
    [map.bounds.se.lat, map.bounds.nw.lng],
    [map.bounds.nw.lat, map.bounds.se.lng],
  ]);

  // boundaries overlay
  const district_boundary = new L.geoJson();
  district_boundary.addTo(map.leaflet);

  fetch("data/StPaulDistrictCouncil.geojson")
    .then((r) => r.json())
    .then((geo) => {
      geo.features.forEach((f) => district_boundary.addData(f));
    })
    .catch((err) => console.log("GeoJSON error:", err));

  // update location + filtering after pan/zoom ENDS
  map.leaflet.on("moveend", handleMoveEnd);

  // set initial location text
  ui.locationText = `${map.center.lat.toFixed(6)}, ${map.center.lng.toFixed(6)}`;
});
</script>

<template>
  <!-- API URL dialog -->
  <dialog id="rest-dialog" ref="dialogRef" open>
    <h1 class="dialog-header">St. Paul Crime REST API</h1>
    <label class="dialog-label">URL:</label>
    <input
      id="dialog-url"
      class="dialog-input"
      type="url"
      v-model="crime_url"
      placeholder="http://localhost:8000"
      required
    />
    <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
    <p class="dialog-error" v-if="ui.apiErr">{{ ui.apiErr }}</p>
    <br />
    <button class="button" type="button" @click="closeDialog">OK</button>
  </dialog>

  <div class="grid-container">
    <!-- Controls -->
    <div class="grid-x grid-padding-x align-middle controls">
      <div class="cell small-12 medium-8">
        <label class="small-label">Location (address or lat,lng)</label>
        <div class="input-group">
          <input class="input-group-field" type="text" v-model="ui.locationText" />
          <div class="input-group-button">
            <button class="button" type="button" @click="goToLocation">Go</button>
          </div>
        </div>
        <p class="inline-error" v-if="ui.locationErr">{{ ui.locationErr }}</p>
      </div>

      <div class="cell small-12 medium-4 text-right">
        <span v-if="ui.loading">Loading…</span>
        <span v-else-if="ui.isReady">
          Showing <b>{{ crimesInView.length }}</b> crimes in view
        </span>
      </div>
    </div>

    <!-- Map -->
    <div class="grid-x grid-padding-x">
      <div id="leafletmap" class="cell"></div>
    </div>

    <!-- Table + Upload -->
    <div class="grid-x grid-padding-x grid-margin-y">
      <div class="cell small-12 medium-7">
        <h2 class="section-title">Crimes (visible neighborhoods only)</h2>

        <table class="hover stack">
          <thead>
            <tr>
              <th>Date</th>
              <th>Time</th>
              <th>Neighborhood</th>
              <th>Incident Type</th>
              <th>Block</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="c in crimesInView" :key="c.case_number">
              <td>{{ c.date }}</td>
              <td>{{ c.time }}</td>
              <td>{{ lookups.nhoodToName.get(c.neighborhood_number) || c.neighborhood_number }}</td>
              <td>{{ lookups.codeToType.get(c.code) || c.code }}</td>
              <td>{{ c.block }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="cell small-12 medium-5">
        <h2 class="section-title">Upload New Incident</h2>

        <div class="callout">
          <label>Case Number
            <input type="text" v-model="upload.case_number" />
          </label>

          <div class="grid-x grid-padding-x">
            <div class="cell small-6">
              <label>Date
                <input type="date" v-model="upload.date" />
              </label>
            </div>
            <div class="cell small-6">
              <label>Time
                <input type="time" step="1" v-model="upload.time" />
              </label>
            </div>
          </div>

          <div class="grid-x grid-padding-x">
            <div class="cell small-6">
              <label>Code
                <input type="number" v-model="upload.code" />
              </label>
            </div>
            <div class="cell small-6">
              <label>Police Grid
                <input type="number" v-model="upload.police_grid" />
              </label>
            </div>
          </div>

          <label>Incident (text)
            <input type="text" v-model="upload.incident" />
          </label>

          <label>Neighborhood Number (1–17)
            <input type="number" v-model="upload.neighborhood_number" />
          </label>

          <label>Block
            <input type="text" v-model="upload.block" />
          </label>

          <p class="inline-error" v-if="upload.err">{{ upload.err }}</p>
          <p class="inline-ok" v-if="upload.ok">{{ upload.ok }}</p>

          <button class="button expanded" type="button" @click="submitIncident" :disabled="upload.submitting">
            {{ upload.submitting ? "Submitting…" : "Submit (PUT /new-incident)" }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
#rest-dialog {
  width: 22rem;
  margin-top: 1rem;
  z-index: 1000;
}

#leafletmap {
  height: 520px;
}

.controls {
  margin-top: 0.75rem;
  margin-bottom: 0.75rem;
}

.section-title {
  font-size: 1.15rem;
  font-weight: 700;
  margin-top: 0.5rem;
}

.small-label {
  font-size: 0.95rem;
  font-weight: 600;
}

.dialog-header {
  font-size: 1.2rem;
  font-weight: bold;
}

.dialog-label {
  font-size: 1rem;
}

.dialog-input {
  font-size: 1rem;
  width: 100%;
}

.dialog-error {
  font-size: 1rem;
  color: #d32323;
}

.inline-error {
  margin: 0.25rem 0 0;
  color: #d32323;
}

.inline-ok {
  margin: 0.25rem 0 0;
  color: #1a7f37;
}
</style>