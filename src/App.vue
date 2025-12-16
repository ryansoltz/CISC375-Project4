<script setup>
import { reactive, ref, computed, onMounted } from "vue";

const crime_url = ref("");
const dialog_err = ref(false);
const dialogRef = ref(null);

const ui = reactive({
  isReady: false,
  loading: false,
  apiErr: "",
  locationText: "",
  locationErr: "",
});

const map = reactive({
  leaflet: null,
  center: { lat: 44.955139, lng: -93.102222 },
  zoom: 12,

  bounds: {
    nw: { lat: 45.008206, lng: -93.217977 },
    se: { lat: 44.883658, lng: -92.993787 },
  },

  

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


const lookups = reactive({
  codeToType: new Map(),
  nhoodToName: new Map(),
});

const typeToCodes = reactive(new Map());

const crimes = ref([]);
const visibleNeighborhoodIds = ref(new Set());

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

const filters = reactive({
  selectedIncidentTypes: new Set(),
  selectedNeighborhoodIds: new Set(),
  startDate: "",
  endDate: "",
  limit: 1000,
});

const selectedCrime = reactive({
  case_number: "",
  marker: null,
});

function clampLatLng(lat, lng) {
  const minLat = map.bounds.se.lat;
  const maxLat = map.bounds.nw.lat;
  const minLng = map.bounds.nw.lng;
  const maxLng = map.bounds.se.lng;

  return {
    lat: Math.min(maxLat, Math.max(minLat, lat)),
    lng: Math.min(maxLng, Math.max(minLng, lng)),
  };
}

function parseLatLng(text) {
  const cleaned = text.trim().replace(/\s+/g, " ");
  const parts = cleaned.includes(",") ? cleaned.split(",") : cleaned.split(" ");
  if (parts.length !== 2) return null;
  const lat = Number(parts[0]);
  const lng = Number(parts[1]);
  if (!Number.isFinite(lat) || !Number.isFinite(lng)) return null;
  return { lat, lng };
}

function splitDateTime(date_time) {
  const dt = String(date_time || "");
  return { date: dt.slice(0, 10), time: dt.slice(11, 19) };
}

async function nominatimSearch(q) {
  const url =
    "https://nominatim.openstreetmap.org/search?" +
    new URLSearchParams({ q, format: "json", limit: "1" });

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
    new URLSearchParams({ lat: String(lat), lon: String(lng), format: "json", zoom: "18" });

  const r = await fetch(url, { headers: { Accept: "application/json" } });
  if (!r.ok) throw new Error("Nominatim reverse failed");
  const data = await r.json();
  return data?.display_name || `${lat.toFixed(6)}, ${lng.toFixed(6)}`;
}

async function fetchCodes() {
  const r = await fetch(`${crime_url.value}/codes`);
  if (!r.ok) throw new Error("Failed /codes");
  const data = await r.json();

  lookups.codeToType.clear();
  typeToCodes.clear();

  for (const row of data) {
    const code = Number(row.code);
    const type = String(row.type);
    lookups.codeToType.set(code, type);

    if (!typeToCodes.has(type)) typeToCodes.set(type, []);
    typeToCodes.get(type).push(code);
  }
}

async function fetchNeighborhoods() {
  const r = await fetch(`${crime_url.value}/neighborhoods`);
  if (!r.ok) throw new Error("Failed /neighborhoods");
  const data = await r.json();

  lookups.nhoodToName.clear();
  filters.selectedNeighborhoodIds.clear();

  for (const row of data) {
    const id = Number(row.id);
    lookups.nhoodToName.set(id, String(row.name));
    filters.selectedNeighborhoodIds.add(id);
  }
}

function buildIncidentQuery() {
  const params = new URLSearchParams();
  params.set("limit", String(filters.limit || 1000));

  if (filters.selectedNeighborhoodIds.size > 0) {
    params.set("neighborhood", Array.from(filters.selectedNeighborhoodIds).join(","));
  }

  if (filters.startDate) params.set("start_date", filters.startDate);
  if (filters.endDate) params.set("end_date", filters.endDate);

  if (filters.selectedIncidentTypes.size > 0) {
    const codes = [];
    for (const t of filters.selectedIncidentTypes) {
      const arr = typeToCodes.get(t) || [];
      for (const c of arr) codes.push(c);
    }
    if (codes.length > 0) params.set("code", codes.join(","));
  }

  return params.toString();
}

async function updateFromFilters() {
  ui.apiErr = "";
  ui.loading = true;
  try {
    const q = buildIncidentQuery();
    const r = await fetch(`${crime_url.value}/incidents?${q}`);
    if (!r.ok) throw new Error("Failed /incidents");

    const data = await r.json();
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

    recomputeVisibleNeighborhoods();
    updateNeighborhoodPopups();
  } catch (e) {
    ui.apiErr = "Failed to update incidents (check API URL + server).";
  } finally {
    ui.loading = false;
  }
}

function recomputeVisibleNeighborhoods() {
  if (!map.leaflet) return;
  const b = map.leaflet.getBounds();
  const visible = new Set();

  for (const nm of map.neighborhood_markers) {
    const [lat, lng] = nm.location;
    if (b.contains([lat, lng])) visible.add(nm.id);
  }
  visibleNeighborhoodIds.value = visible;
}

const crimesInView = computed(() => {
  const visible = visibleNeighborhoodIds.value;
  if (!visible || visible.size === 0) return crimes.value; 
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

function initializeNeighborhoodMarkers() {
  for (const nm of map.neighborhood_markers) {
    if (nm.marker) continue;
    nm.marker = L.marker(nm.location).addTo(map.leaflet);
  }
  updateNeighborhoodPopups();
}

function updateNeighborhoodPopups() {
  const counts = crimeCountsByNeighborhood.value;
  for (const nm of map.neighborhood_markers) {
    if (!nm.marker) continue;
    const name = lookups.nhoodToName.get(nm.id) || `Neighborhood ${nm.id}`;
    const count = counts.get(nm.id) || 0;
    nm.marker.bindPopup(`<b>${name}</b><br/>Crimes in view: ${count}`);
  }
}

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
      lat = parsed.lat;
      lng = parsed.lng;
    } else {
      const hit = await nominatimSearch(text);
      if (!hit) {
        ui.locationErr = "No results found.";
        return;
      }
      lat = hit.lat;
      lng = hit.lng;
    }

    const clamped = clampLatLng(lat, lng);
    map.leaflet.setView([clamped.lat, clamped.lng], Math.max(map.leaflet.getZoom(), 12));
    ui.locationText = `${clamped.lat.toFixed(6)}, ${clamped.lng.toFixed(6)}`;
  } catch {
    ui.locationErr = "Could not geocode location.";
  }
}

async function handleMoveEnd() {
  if (!map.leaflet) return;

  const c = map.leaflet.getCenter();
  const clamped = clampLatLng(c.lat, c.lng);

  if (clamped.lat !== c.lat || clamped.lng !== c.lng) {
    map.leaflet.panTo([clamped.lat, clamped.lng], { animate: false });
  }

  recomputeVisibleNeighborhoods();
  updateNeighborhoodPopups();

  try {
    ui.locationText = await nominatimReverse(clamped.lat, clamped.lng);
  } catch {
    ui.locationText = `${clamped.lat.toFixed(6)}, ${clamped.lng.toFixed(6)}`;
  }
}

function crimeCategory(c) {
  const t = (lookups.codeToType.get(c.code) || "").toUpperCase();
  const i = (c.incident || "").toUpperCase();

  const violentHints = ["ASSAULT", "ROBBERY", "RAPE", "HOMICIDE", "MURDER", "KIDNAP", "WEAPON", "DOMESTIC"];
  if (violentHints.some((h) => t.includes(h) || i.includes(h))) return "violent";

  const propertyHints = ["BURGLARY", "THEFT", "LARCENY", "AUTO THEFT", "VANDAL", "ARSON", "FRAUD", "SHOPLIFT"];
  if (propertyHints.some((h) => t.includes(h) || i.includes(h))) return "property";

  return "other";
}

function rowClass(c) {
  const cat = crimeCategory(c);
  return {
    "row-violent": cat === "violent",
    "row-property": cat === "property",
    "row-other": cat === "other",
  };
}

async function deleteIncident(case_number) {
  if (!case_number) return;
  ui.apiErr = "";
  try {
    const r = await fetch(`${crime_url.value}/remove-incident`, {
      method: "DELETE",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ case_number }),
    });

    if (!r.ok) {
      const txt = await r.text().catch(() => "");
      throw new Error(txt || "Delete failed");
    }

    if (selectedCrime.case_number === case_number && selectedCrime.marker) {
      map.leaflet.removeLayer(selectedCrime.marker);
      selectedCrime.marker = null;
      selectedCrime.case_number = "";
    }

    await updateFromFilters();
  } catch {
    ui.apiErr = `Delete failed for ${case_number}.`;
  }
}

function normalizeBlockForGeocode(block) {
  let b = String(block || "").trim();
  b = b.replace(/^(\d+)X\b/, (_, digits) => `${digits}0`);
  return b;
}

async function selectCrime(c) {
  if (!map.leaflet) return;

  if (selectedCrime.marker) {
    map.leaflet.removeLayer(selectedCrime.marker);
    selectedCrime.marker = null;
  }

  selectedCrime.case_number = c.case_number;

  const query = `${normalizeBlockForGeocode(c.block)}, Saint Paul, MN`;
  try {
    const hit = await nominatimSearch(query);
    if (!hit) return;

    const icon = L.divIcon({
      className: "selected-crime-icon",
      html: '<div class="selected-dot"></div>',
      iconSize: [16, 16],
      iconAnchor: [8, 8],
    });

    selectedCrime.marker = L.marker([hit.lat, hit.lng], { icon }).addTo(map.leaflet);

    const popupId = `del-${c.case_number}`;
    const html = `
      <div>
        <b>${c.date} ${c.time}</b><br/>
        ${c.incident}<br/>
        <small>${c.block}</small><br/>
        <button id="${popupId}" class="button tiny alert" type="button">Delete</button>
      </div>
    `;

    selectedCrime.marker.bindPopup(html).openPopup();

    selectedCrime.marker.on("popupopen", () => {
      const btn = document.getElementById(popupId);
      if (btn) btn.onclick = () => deleteIncident(c.case_number);
    });

    map.leaflet.panTo([hit.lat, hit.lng]);
  } catch {
  }
}

async function initializeCrimes() {
  ui.apiErr = "";
  ui.loading = true;
  try {
    await fetchCodes();
    await fetchNeighborhoods();
    filters.limit = 1000;
    await updateFromFilters();

    initializeNeighborhoodMarkers();
    await handleMoveEnd();

    ui.isReady = true;
  } catch {
    ui.apiErr = "Could not load data from Crime API. Check URL and server.";
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

async function submitIncident() {
  upload.err = "";
  upload.ok = "";

  const required = [
    upload.case_number,
    upload.date,
    upload.time,
    upload.code,
    upload.incident,
    upload.police_grid,
    upload.neighborhood_number,
    upload.block,
  ];

  if (required.some((v) => String(v).trim() === "")) {
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
      upload.err = payload?.error || "Upload failed.";
      return;
    }

    upload.ok = `Uploaded incident ${body.case_number}.`;
    
    await updateFromFilters();

  } catch (e) {
    upload.err = "Upload failed (server/network error).";
  } finally {
    upload.submitting = false;
  }
}

onMounted(() => {
  map.leaflet = L.map("leafletmap").setView([map.center.lat, map.center.lng], map.zoom);

  L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    minZoom: 11,
    maxZoom: 18,
  }).addTo(map.leaflet);

  map.leaflet.setMaxBounds([
    [map.bounds.se.lat, map.bounds.nw.lng],
    [map.bounds.nw.lat, map.bounds.se.lng],
  ]);

  const district_boundary = new L.geoJson();
  district_boundary.addTo(map.leaflet);

  fetch("data/StPaulDistrictCouncil.geojson")
    .then((r) => r.json())
    .then((geo) => geo.features.forEach((f) => district_boundary.addData(f)))
    .catch((err) => console.log("GeoJSON error:", err));

  map.leaflet.on("moveend", handleMoveEnd);

  ui.locationText = `${map.center.lat.toFixed(6)}, ${map.center.lng.toFixed(6)}`;
});
</script>

<template>
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
    <div class="grid-x grid-padding-x align-middle">
      <div class="cell medium-8">
        <h1>St. Paul Crime Map</h1>
      </div>
      <div class="cell medium-2">
        <a ref="">Crime Map</a>
      </div>
      <div class="cell medium-2">
        <a href="about.html">About</a>
      </div>      
    </div>
  </div>
</br>
  <div class="grid-container">
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

      </div>
    </div>

    <div class="grid-x grid-padding-x grid-margin-y">
      <div class="cell small-12 medium-4">
        <h2 class="section-title">Filters</h2>

        <label>Max incidents</label>
        <select v-model.number="filters.limit">
          <option :value="250">250</option>
          <option :value="500">500</option>
          <option :value="1000">1000</option>
          <option :value="2000">2000</option>
        </select>

        <div class="grid-x grid-padding-x">
          <div class="cell small-6">
            <label>Start date</label>
            <input type="date" v-model="filters.startDate" />
          </div>
          <div class="cell small-6">
            <label>End date</label>
            <input type="date" v-model="filters.endDate" />
          </div>
        </div>

        <button class="button expanded" type="button" @click="updateFromFilters">
          Update
        </button>

        <p class="inline-error" v-if="ui.apiErr">{{ ui.apiErr }}</p>
      </div>

      <div class="cell small-12 medium-4">
        <h3 class="sub-title">Neighborhoods</h3>
        <div class="scroll-box">
          <label v-for="[id, name] in Array.from(lookups.nhoodToName.entries())" :key="id">
            <input
              type="checkbox"
              :checked="filters.selectedNeighborhoodIds.has(id)"
              @change="(e) => {
                if (e.target.checked) filters.selectedNeighborhoodIds.add(id);
                else filters.selectedNeighborhoodIds.delete(id);
              }"
            />
            {{ name }}
          </label>
        </div>
      </div>

      <div class="cell small-12 medium-4">
        <h3 class="sub-title">Incident Types</h3>
        <div class="scroll-box">
          <label v-for="t in Array.from(typeToCodes.keys()).sort()" :key="t">
            <input
              type="checkbox"
              :checked="filters.selectedIncidentTypes.has(t)"
              @change="(e) => {
                if (e.target.checked) filters.selectedIncidentTypes.add(t);
                else filters.selectedIncidentTypes.delete(t);
              }"
            />
            {{ t }}
          </label>
        </div>
      </div>
    </div>

    <div class="grid-x grid-padding-x">
      <div id="leafletmap" class="cell"></div>
    </div>

    <div class="grid-x grid-padding-x grid-margin-y">
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
            <input type="text" v-model="upload.block" placeholder="79X 6 ST E" />
          </label>

          <p class="inline-error" v-if="upload.err">{{ upload.err }}</p>
          <p class="inline-ok" v-if="upload.ok">{{ upload.ok }}</p>

          <button class="button expanded" type="button" @click="submitIncident" :disabled="upload.submitting">
            {{ upload.submitting ? "Submitting…" : "Submit" }}
          </button>
        </div>
      </div>
    </div>

    <div class="grid-x grid-padding-x grid-margin-y">
      <div class="cell small-12">
        <h2 class="section-title">Crimes (click a row to drop an exact marker)</h2>
        <div class="legend">
          <span class="legend-item legend-violent">Violent</span>
          <span class="legend-item legend-property">Property</span>
          <span class="legend-item legend-other">Other</span>
        </div>

        <table class="hover stack">
          <thead>
            <tr>
              <th>Date</th>
              <th>Time</th>
              <th>Neighborhood</th>
              <th>Incident Type</th>
              <th>Block</th>
              <th>Delete</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="c in crimesInView"
              :key="c.case_number"
              :class="rowClass(c)"
              @click="selectCrime(c)"
              style="cursor: pointer;"
            >
              <td>{{ c.date }}</td>
              <td>{{ c.time }}</td>
              <td>{{ lookups.nhoodToName.get(c.neighborhood_number) || c.neighborhood_number }}</td>
              <td>{{ lookups.codeToType.get(c.code) || c.code }}</td>
              <td>{{ c.block }}</td>
              <td>
                <button class="button tiny alert" type="button" @click.stop="deleteIncident(c.case_number)">
                  Delete
                </button>
              </td>
            </tr>
          </tbody>
        </table>


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
  font-family: Tahoma; 
  font-size: 1.15rem;
  font-weight: 700;
  margin-top: 0.5rem;
}

.small-label {
  font-family: Tahoma; 
  font-size: 0.95rem;
  font-weight: 600;
}

.dialog-header {
  font-family: Tahoma; 
  font-size: 1.2rem;
  font-weight: bold;
}

.dialog-label {
  font-size: 1rem;
}

.dialog-input {
  font-family: Tahoma; 
  font-size: 1rem;
  width: 100%;
}

.dialog-error {
  font-family: Tahoma; 
  font-size: 1rem;
  color: #d32323;
}

.inline-error {
  font-family: Tahoma; 
  margin: 0.25rem 0 0;
  color: #d32323;
}

.scroll-box {
  max-height: 240px;
  overflow: auto;
  border: 1px solid #ddd;
  padding: 0.5rem;
  background: #fff;
}

.sub-title {
  font-family: Tahoma; 
  font-size: 1rem;
  font-weight: 700;
  margin-bottom: 0.25rem;
}

.legend {
  display: flex;
  gap: 0.5rem;
  margin: 0.5rem 0;
}

.legend-item {
  font-family: Tahoma; 
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  font-weight: 700;
  font-size: 0.9rem;
}

.legend-violent {
  background: #ffd6d6;
}
.legend-property {
  background: #d6e8ff;
}
.legend-other {
  background: #e6e6e6;
}

.row-violent td {
  background: #ffd6d6;
}
.row-property td {
  background: #d6e8ff;
}
.row-other td {
  background: #e6e6e6;
}

.selected-crime-icon .selected-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 2px solid #fff;
  background: #d32323;
  box-shadow: 0 0 6px rgba(0, 0, 0, 0.4);
}

.hint {
  font-family: Tahoma; 
  margin-top: 0.5rem;
  font-size: 0.9rem;
  color: #555;
}


</style>