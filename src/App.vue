<script setup>
import { computed, onMounted, ref, watch } from 'vue'

const STORAGE_KEY = 'dof-calculator-saved-configurations'
const THEME_STORAGE_KEY = 'dof-calculator-theme'

const cocOptions = [
  { label: 'Full-frame (0.029 mm)', value: '0.029' },
  { label: 'APS-C (0.018 mm)', value: '0.018' },
  { label: 'Micro four thirds (0.015 mm)', value: '0.015' },
  { label: '1 inch (0.011 mm)', value: '0.011' },
]

const circleOfConfusion = ref('0.029')
const focalLength = ref(50)
const aperture = ref(2.8)
const subjectDistance = ref(3)
const savedConfigurations = ref([])
const theme = ref('dark')

const subjectDistanceMm = computed(() => Math.max(Number(subjectDistance.value) || 0, 0) * 1000)
const focalLengthMm = computed(() => Math.max(Number(focalLength.value) || 0, 0.1))
const apertureValue = computed(() => Math.max(Number(aperture.value) || 0, 0.1))
const cocValue = computed(() => Math.max(Number(circleOfConfusion.value) || 0, 0.001))

const approximateDofMm = computed(() => {
  const u = subjectDistanceMm.value
  const f = focalLengthMm.value
  const n = apertureValue.value
  const c = cocValue.value

  return (2 * u * u * n * c) / (f * f)
})

const hyperfocalMm = computed(() => {
  const f = focalLengthMm.value
  const n = apertureValue.value
  const c = cocValue.value

  return (f * f) / (n * c) + f
})

const nearLimitMm = computed(() => {
  const h = hyperfocalMm.value
  const u = subjectDistanceMm.value
  const f = focalLengthMm.value

  return (h * u) / (h + (u - f))
})

const farLimitMm = computed(() => {
  const h = hyperfocalMm.value
  const u = subjectDistanceMm.value
  const f = focalLengthMm.value

  if (u >= h) {
    return Number.POSITIVE_INFINITY
  }

  return (h * u) / (h - (u - f))
})

const dofDisplay = computed(() => {
  if (!Number.isFinite(farLimitMm.value)) {
    return 'Infinity'
  }

  return formatDistance(approximateDofMm.value)
})

const nearDisplay = computed(() => formatDistance(nearLimitMm.value))
const farDisplay = computed(() =>
  Number.isFinite(farLimitMm.value) ? formatDistance(farLimitMm.value) : 'Infinity'
)
const cocDisplay = computed(() =>
  cocValue.value.toFixed(3).replace(/0+$/, '').replace(/\.$/, '')
)
const themeSymbol = computed(() => (theme.value === 'dark' ? '☀' : '☾'))
const themeLabel = computed(() => (theme.value === 'dark' ? 'Light theme' : 'Dark theme'))

onMounted(() => {
  const storedTheme = window.localStorage.getItem(THEME_STORAGE_KEY)

  if (storedTheme === 'light' || storedTheme === 'dark') {
    theme.value = storedTheme
  }

  const stored = window.localStorage.getItem(STORAGE_KEY)

  if (!stored) {
    return
  }

  try {
    const parsed = JSON.parse(stored)
    savedConfigurations.value = Array.isArray(parsed) ? parsed : []
  } catch {
    savedConfigurations.value = []
  }
})

watch(
  savedConfigurations,
  (value) => {
    window.localStorage.setItem(STORAGE_KEY, JSON.stringify(value))
  },
  { deep: true },
)

watch(
  theme,
  (value) => {
    document.documentElement.setAttribute('data-theme', value)
    window.localStorage.setItem(THEME_STORAGE_KEY, value)
  },
  { immediate: true },
)

function saveConfiguration() {
  savedConfigurations.value.unshift({
    id: crypto.randomUUID(),
    coc: cocDisplay.value,
    focalLength: Number(focalLength.value) || 0,
    aperture: Number(aperture.value) || 0,
    subjectDistance: Number(subjectDistance.value) || 0,
    dof: dofDisplay.value,
  })
}

function deleteConfiguration(id) {
  savedConfigurations.value = savedConfigurations.value.filter((configuration) => configuration.id !== id)
}

function toggleTheme() {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}

function formatDistance(distanceMm) {
  const distanceMeters = distanceMm / 1000

  if (distanceMeters >= 100) {
    return `${distanceMeters.toFixed(1)} m`
  }

  if (distanceMeters >= 10) {
    return `${distanceMeters.toFixed(2)} m`
  }

  return `${distanceMeters.toFixed(3)} m`
}
</script>

<template>
  <main class="shell">
    <div class="topbar">
      <button class="theme-toggle" type="button" @click="toggleTheme">
        <span class="theme-toggle-symbol" aria-hidden="true">{{ themeSymbol }}</span>
        <span>{{ themeLabel }}</span>
      </button>
    </div>

    <section class="hero-panel">
      <p class="eyebrow">Calculate depth of field</p>
      <h1>Depth of Field</h1>
      <p class="intro">
        Estimate depth of field from focal length, aperture, subject distance, and your
        selected circle of confusion value.
      </p>

      <div class="result-card">
        <p class="result-label">Depth of field</p>
        <p class="result-value">{{ dofDisplay }}</p>
        <p class="result-note">Approximate total depth of field in meters</p>
      </div>

      <div class="range-grid" aria-label="Depth of field limits">
        <article class="range-card">
          <p class="range-label">Near limit</p>
          <p class="range-value">{{ nearDisplay }}</p>
        </article>
        <article class="range-card">
          <p class="range-label">Far limit</p>
          <p class="range-value">{{ farDisplay }}</p>
        </article>
      </div>
    </section>

    <section class="control-panel">
      <div class="panel-header">
        <h2>Inputs</h2>
        <p>Start with circle of confusion, then fine-tune your lens and distance settings.</p>
      </div>

      <div class="form-grid">
        <label class="field">
          <span>Circle of confusion</span>
          <select v-model="circleOfConfusion">
            <option
              v-for="option in cocOptions"
              :key="option.value"
              :value="option.value"
            >
              {{ option.label }}
            </option>
          </select>
        </label>

        <label class="field">
          <span>Focal length</span>
          <div class="input-wrap">
            <input v-model="focalLength" type="number" min="1" step="1" />
            <small>mm</small>
          </div>
        </label>

        <label class="field">
          <span>Aperture</span>
          <div class="input-wrap">
            <input v-model="aperture" type="number" min="0.7" step="0.1" />
            <small>f</small>
          </div>
        </label>

        <label class="field">
          <span>Subject distance</span>
          <div class="input-wrap">
            <input v-model="subjectDistance" type="number" min="0.1" step="0.1" />
            <small>m</small>
          </div>
        </label>
      </div>

      <button class="save-button" type="button" @click="saveConfiguration">
        Save configuration
      </button>

      <div class="formula-card">
        <p class="formula-heading">Depth of field formula</p>
        <p class="formula-line">DOF ≈ (2 × u² × N × c) / f²</p>
        <p class="formula-copy">
          Using a circle of confusion of <strong>{{ cocDisplay }} mm</strong>.
        </p>
      </div>
    </section>

    <section class="saved-panel">
      <div class="panel-header">
        <h2>Saved Configurations</h2>
        <p>Compare saved setups side by side and remove the ones you no longer need.</p>
      </div>

      <div v-if="savedConfigurations.length" class="saved-list" aria-label="Saved configurations">
        <div class="saved-table saved-table-head" aria-hidden="true">
          <span>CoC</span>
          <span>Focal</span>
          <span>Aperture</span>
          <span>Distance</span>
          <span>DoF</span>
          <span>Delete</span>
        </div>

        <article
          v-for="configuration in savedConfigurations"
          :key="configuration.id"
          class="saved-table saved-row"
        >
          <span><small>CoC</small>{{ configuration.coc }} mm</span>
          <span><small>Focal</small>{{ configuration.focalLength }} mm</span>
          <span><small>Aperture</small>f/{{ configuration.aperture }}</span>
          <span><small>Distance</small>{{ configuration.subjectDistance }} m</span>
          <span><small>DoF</small>{{ configuration.dof }}</span>
          <button
            class="delete-button"
            type="button"
            @click="deleteConfiguration(configuration.id)"
          >
            Delete
          </button>
        </article>
      </div>

      <div v-else class="empty-state">
        <p>No saved configurations yet.</p>
        <p>Use “Save configuration” to build up a comparison list.</p>
      </div>
    </section>
  </main>
</template>
