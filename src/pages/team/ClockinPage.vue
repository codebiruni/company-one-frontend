<template>
  <div class="q-pa-md">
    <!-- Loading -->
    <div v-if="loading" class="flex flex-center q-pa-xl">
      <q-spinner-dots size="50px" color="primary" />
    </div>

    <template v-else>
      <!-- Tabs -->
      <q-tabs v-model="tab" dense class="q-mb-md" align="left" active-color="primary" indicator-color="primary">
        <q-tab name="clock" icon="schedule" label="Entry/Exit" />
        <q-tab name="reports" icon="description" label="Daily Report" />
      </q-tabs>

      <!-- Clock In / Out tab -->
      <div v-if="tab === 'clock'" class="row justify-center q-col-gutter-md">
        <div class="col-12 col-md-7 col-lg-6">

          <!-- Header / Summary -->
          <q-card class="q-mb-md">
            <q-card-section class="row items-center no-wrap">
              <div class="col">
                <div class="text-h6">Today</div>
                <div class="text-caption text-grey-7">{{ todayISO }}</div>
              </div>

              <q-chip v-if="status === 'IN'" color="positive" text-color="white" icon="login" square>
                Clocked In
              </q-chip>

              <q-chip v-else-if="status === 'OUT'" color="grey-8" text-color="white" icon="task_alt" square>
                Completed
              </q-chip>

              <q-chip v-else color="warning" text-color="black" icon="schedule" square>
                Not Clocked In
              </q-chip>
            </q-card-section>

            <q-separator />

            <q-card-section class="row q-col-gutter-md">
              <div class="col-12 col-sm-6">
                <q-item dense>
                  <q-item-section avatar>
                    <q-icon name="login" color="positive" />
                  </q-item-section>
                  <q-item-section>
                    <q-item-label>Clock In</q-item-label>
                    <q-item-label caption>
                      {{ todayRecord?.clockin_time ? formatTime(todayRecord.clockin_time) : '—' }}
                    </q-item-label>
                  </q-item-section>
                </q-item>

                <q-item dense>
                  <q-item-section avatar>
                    <q-icon name="logout" color="warning" />
                  </q-item-section>
                  <q-item-section>
                    <q-item-label>Clock Out</q-item-label>
                    <q-item-label caption>
                      {{ todayRecord?.clockout_time ? formatTime(todayRecord.clockout_time) : '—' }}
                    </q-item-label>
                  </q-item-section>
                </q-item>
              </div>

              <div class="col-12 col-sm-6">
                <q-item dense>
                  <q-item-section avatar>
                    <q-icon name="timer" color="primary" />
                  </q-item-section>
                  <q-item-section>
                    <q-item-label>Worked</q-item-label>
                    <q-item-label caption>
                      <span class="text-weight-medium">{{ workedHHMMSS }}</span>
                      <span v-if="status === 'IN'" class="text-grey-7"> (running)</span>
                    </q-item-label>
                  </q-item-section>
                </q-item>

                <q-item dense>
                  <q-item-section avatar>
                    <q-icon name="pin_drop" color="grey-7" />
                  </q-item-section>
                  <q-item-section>
                    <q-item-label>Location</q-item-label>
                    <q-item-label caption class="ellipsis-2-lines">
                      {{ activeLocation || '—' }}
                    </q-item-label>
                  </q-item-section>
                </q-item>
              </div>
            </q-card-section>
          </q-card>

          <!-- Clock In -->
          <q-card v-if="status === 'NONE'" class="q-mb-md">
            <q-card-section class="row items-center">
              <div class="text-h6">Clock In</div>
              <q-space />
              <q-btn flat dense icon="my_location" label="Refresh" @click="refreshInMeta" />
            </q-card-section>

            <q-separator />

            <q-card-section>
              <q-form @submit.prevent="clockIn">
                <div class="row q-col-gutter-md q-mb-md">
                  <div class="col-6">
                    <q-input v-model="clockInData.time" label="Time" readonly outlined />
                  </div>
                  <div class="col-6">
                    <q-select v-model="clockInData.travel" :options="TRAVEL_OPTIONS" label="Travel Time" outlined
                      emit-value map-options />
                  </div>
                  <div class="col-12">
                    <q-input v-model="clockInData.location" label="Location" readonly outlined />
                  </div>
                </div>

                <q-file v-model="clockInPhotos" label="Worksite Photos (min 3)" multiple accept="image/*" outlined
                  counter max-files="10" @update:model-value="handleClockInPhotos" />

                <q-btn type="submit" label="Clock In" color="primary" unelevated :loading="submitting"
                  class="full-width" />
              </q-form>
            </q-card-section>
          </q-card>

          <!-- Clock Out -->
          <q-card v-else-if="status === 'IN'" class="q-mb-md">
            <q-card-section class="row items-center">
              <div class="text-h6">Clock Out</div>
              <q-space />
              <q-btn flat dense icon="my_location" label="Refresh" @click="refreshOutMeta" />
            </q-card-section>

            <q-separator />

            <q-card-section>
              <q-form @submit.prevent="clockOut">
                <div class="row q-col-gutter-md q-mb-md">
                  <div class="col-6">
                    <q-input v-model="clockOutData.time" label="Time" readonly outlined />
                  </div>
                  <div class="col-6">
                    <q-input v-model="clockOutData.travel" label="Travel Time" outlined readonly />
                  </div>

                  <div class="col-12">
                    <q-input v-model="clockOutData.location" label="Location" readonly outlined />
                  </div>
                </div>

                <q-file v-model="clockOutPhotos" label="Worksite Photos (min 3)" multiple accept="image/*" outlined
                  counter max-files="10" @update:model-value="handleClockOutPhotos" />

                <q-file v-model="clockOutReceipts" label="Receipts (optional)" multiple accept="image/*" outlined
                  counter @update:model-value="handleClockOutReceipts" />

                <q-toggle v-model="clockOutData.hadBreak" label="Had break today" />

                <q-btn type="submit" label="Clock Out" color="warning" unelevated :loading="submitting"
                  class="full-width" />
              </q-form>
            </q-card-section>
          </q-card>

          <!-- Completed -->
          <q-card v-else class="q-mb-md">
            <q-card-section class="text-center">
              <q-icon name="task_alt" size="48px" color="positive" />
              <div class="text-h6 q-mt-sm">Shift Completed</div>
              <div class="text-caption text-grey-7">
                Total worked: <span class="text-weight-medium">{{ workedHHMMSS }}</span>
              </div>
            </q-card-section>
          </q-card>

        </div>
      </div>

      <!-- Daily Report tab -->
      <div v-else class="row justify-center q-col-gutter-md">

        <!-- Management: team-wide view -->
        <div v-if="isManagement" class="col-12">
          <!-- Toolbar (desktop, teleported to the layout header) -->
          <teleport to=".toolbar-actions" v-if="isMounted">
            <div class="reports-toolbar-actions gt-sm row q-col-gutter-sm items-center">
              <div style="min-width:150px;">
                <q-select v-model="selectedStaff" :options="staffOptions" label="Staff" dense outlined clearable
                  emit-value map-options />
              </div>
              <div>
                <q-input v-model="monthView" type="date" label="Month" dense outlined />
              </div>
              <div>
                <q-btn label="Filter" class="q-pa-sm" size="md" color="primary" @click="fetchTeamReports" />
              </div>
              <div>
                <q-btn label="Export PDF" color="secondary" @click="exportTeamPDF" />
              </div>
              <div>
                <q-btn unelevated color="primary" icon="add" label="New report" @click="openTeamCreateDialog" />
              </div>
            </div>
          </teleport>

          <!-- Mobile filters -->
          <div class="reports-mobile-filters lt-md">
            <div class="reports-filter-grid">
              <div>
                <q-select v-model="selectedStaff" :options="staffOptions" label="Staff" dense outlined clearable
                  emit-value map-options />
              </div>
              <div>
                <q-input v-model="monthView" type="date" label="Month" dense outlined />
              </div>
              <q-btn label="Filter" class="q-pa-sm" color="primary" @click="fetchTeamReports" />
              <q-btn label="Export PDF" color="secondary" @click="exportTeamPDF" />
              <q-btn unelevated color="primary" icon="add" label="New report" @click="openTeamCreateDialog" />
            </div>
          </div>

          <!-- Totals -->
          <div class="reports-summary-grid q-mb-xl">
            <q-card class="reports-summary-card" flat bordered>
              <q-card-section class="reports-summary-card__body">
                <div class="reports-summary-card__label">Total Visits</div>
                <q-badge class="text-h6" color="secondary">{{ teamReports.length }}</q-badge>
              </q-card-section>
            </q-card>

            <q-card class="reports-summary-card" flat bordered>
              <q-card-section class="reports-summary-card__body">
                <div class="reports-summary-card__label">Interested</div>
                <q-badge class="text-h6" color="green">{{ teamCountByStatus('interested') }}</q-badge>
              </q-card-section>
            </q-card>

            <q-card class="reports-summary-card" flat bordered>
              <q-card-section class="reports-summary-card__body">
                <div class="reports-summary-card__label">Follow Ups</div>
                <q-badge class="text-h6" color="orange">{{ teamCountByStatus('follow_up') }}</q-badge>
              </q-card-section>
            </q-card>

            <q-card class="reports-summary-card" flat bordered>
              <q-card-section class="reports-summary-card__body">
                <div class="reports-summary-card__label">Not Interested</div>
                <q-badge class="text-h6" color="red">{{ teamCountByStatus('not_interested') }}</q-badge>
              </q-card-section>
            </q-card>
          </div>

          <!-- Team reports table -->
          <q-table :rows="teamReports" :columns="teamColumns" row-key="id" flat bordered :loading="teamLoading">

            <template v-slot:body="props">
              <q-tr :props="props">

                <q-td class="text-center">
                  {{ formatDate(props.row.date) }}
                </q-td>

                <q-td class="text-center">
                  {{ props.row.staffName }}
                </q-td>

                <q-td class="text-center">
                  {{ props.row.institution }}
                </q-td>

                <q-td class="text-center">
                  {{ props.row.phone || '—' }}
                </q-td>

                <q-td class="text-center">
                  {{ props.row.email || '—' }}
                </q-td>

                <q-td class="text-center">
                  <q-chip :color="statusColor(props.row.status)" text-color="white" size="sm" square>
                    {{ statusLabel(props.row.status) }}
                  </q-chip>
                </q-td>

                <q-td class="text-left" style="max-width: 260px;">
                  <div class="ellipsis-3-lines">{{ props.row.comments || '—' }}</div>
                </q-td>

                <q-td class="text-center">
                  <div>
                    <q-img v-for="(url, i) in props.row.photo_urls" :key="i" :src="url" spinner-color="primary"
                      style="width:50px;height:50px;object-fit:cover" />
                  </div>
                </q-td>

                <q-td class="text-center">
                  <q-btn flat round dense icon="edit" color="primary" @click="openTeamEditDialog(props.row)">
                    <q-tooltip>Edit report</q-tooltip>
                  </q-btn>
                </q-td>

              </q-tr>
            </template>

          </q-table>
        </div>
      </div>
    </template>

    <!-- Report Dialog -->
    <q-dialog v-model="reportDialog.open">
      <q-card style="width: 560px; max-width: 95vw;">
        <q-card-section class="row items-center">
          <div class="text-h6">{{ reportDialog.id ? 'Edit Visit' : 'Add Visit' }}</div>
          <q-space />
          <q-btn flat round dense icon="close" v-close-popup />
        </q-card-section>
        <q-separator />
        <q-card-section>
          <q-form @submit.prevent="saveReport">
            <div class="row q-col-gutter-md">
              <div class="col-12">
                <q-input v-model="reportForm.institution" label="Institution / School name" outlined dense
                  :rules="[val => !!val || 'Institution name is required']" />
              </div>
              <div class="col-6">
                <q-input v-model="reportForm.phone" label="Phone" outlined dense />
              </div>
              <div class="col-6">
                <q-input v-model="reportForm.email" label="Email" type="email" outlined dense />
              </div>
              <div class="col-12">
                <q-select v-model="reportForm.status" :options="STATUS_OPTIONS" label="Status" outlined dense emit-value
                  map-options />
              </div>
              <div class="col-12">
                <q-input v-model="reportForm.location" label="Location" outlined dense />
              </div>
              <div class="col-12">
                <q-input v-model="reportForm.comments" label="Comments" type="textarea" outlined dense autogrow />
              </div>
              <div class="col-12">
                <q-file v-model="reportPhotos" label="Photos (optional)" multiple accept="image/*" outlined counter
                  max-files="10" @update:model-value="handleReportPhotos" />
              </div>
            </div>
            <div class="row q-mt-md justify-end q-gutter-sm">
              <q-btn flat label="Cancel" v-close-popup />
              <q-btn type="submit" label="Save" color="primary" unelevated :loading="reportSaving" />
            </div>
          </q-form>
        </q-card-section>
      </q-card>
    </q-dialog>

    <!-- Management Create / Edit Dialog -->
    <DailyReportFormDialog v-model="teamReportDialog.open" :report="teamReportDialog.report"
      @saved="onTeamReportSaved" />
  </div>
</template>

<script setup lang="ts">
import { ref, reactive, computed, onMounted, onBeforeUnmount } from 'vue'
import { Notify } from 'quasar'
import imageCompression from 'browser-image-compression'
import { pb, baseUrl } from 'src/services/pocketbase'
import {
  Collections,
  type TimesheetRecord,
  type DailyReportsRecord,
  type DailyReportsResponse,
  type DailyReportsStatusOptions,
  type UsersResponse
} from 'src/types/pocketbase-types'
import { formatDate } from 'src/services/utils'

import DailyReportFormDialog from './DailyReportFormDialog.vue'

import jsPDF from 'jspdf'
import autoTable from 'jspdf-autotable'

interface ClockInData {
  time: string
  location: string
  travel: string
  photos: File[]
}

interface ClockOutData {
  time: string
  location: string
  travel: string
  photos: File[]
  receipts: File[]
  hadBreak: boolean
}

const TRAVEL_OPTIONS = [
  { label: '1H30', value: '1H30' },
  { label: '2H00', value: '2H00' },
  { label: '2H30', value: '2H30' },
  { label: '3H00', value: '3H00' },
  { label: '3H30', value: '3H30' },
  { label: '4H00', value: '4H00' }
]

const STATUS_OPTIONS = [
  { label: 'Visited', value: 'visited' },
  { label: 'Interested', value: 'interested' },
  { label: 'Not Interested', value: 'not_interested' },
  { label: 'Follow Up', value: 'follow_up' },
  { label: 'Closed', value: 'closed' }
]

const loading = ref(true)
const submitting = ref(false)

const todayRecord = ref<TimesheetRecord | null>(null)

const clockInData = ref<ClockInData>({
  time: '',
  location: '',
  travel: '',
  photos: []
})

const clockOutData = ref<ClockOutData>({
  time: '',
  location: '',
  travel: '',
  photos: [],
  receipts: [],
  hadBreak: true
})

const clockInPhotos = ref<File[]>([])
const clockOutPhotos = ref<File[]>([])
const clockOutReceipts = ref<File[]>([])

const reports = ref<DailyReportsRecord[]>([])
const reportDate = ref(new Date().toISOString().slice(0, 10))
const reportSaving = ref(false)
const reportPhotos = ref<File[]>([])

const reportDialog = reactive({
  open: false,
  id: null as string | null
})

const reportForm = reactive({
  institution: '',
  phone: '',
  email: '',
  status: 'visited' as DailyReportsStatusOptions,
  comments: '',
  location: '',
  photos: [] as File[]
})

const currentUserId = computed(() => pb.authStore.record?.id)

const todayISO = computed(() => new Date().toISOString().slice(0, 10))

const status = computed<'NONE' | 'IN' | 'OUT'>(() => {
  if (!todayRecord.value?.clockin_time) return 'NONE'
  if (todayRecord.value?.clockin_time && !todayRecord.value?.clockout_time) return 'IN'
  return 'OUT'
})

const activeLocation = computed(() => {
  if (status.value === 'IN') return todayRecord.value?.clockin_location || clockOutData.value.location
  if (status.value === 'OUT') return todayRecord.value?.clockout_location || ''
  return clockInData.value.location
})

const formatTime = (time?: string) => {
  if (!time) return ''
  return new Date(time).toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' })
}

/**
 * Timer / Worked time (SaaS-friendly)
 * - Source of truth: ISO times in PocketBase (clockin_time/clockout_time).
 * - UI updates: an interval ticks every second while clocked in.
 * - Avoid relying on clockInData.time/clockOutData.time for calculations (those are display-only).
 */
const tick = ref(0)
let intervalId: number | undefined

function pad2(n: number) {
  return String(n).padStart(2, '0')
}

const workedMs = computed(() => {
  // force recompute while running
  void tick.value

  const cin = todayRecord.value?.clockin_time
  if (!cin) return 0

  const start = new Date(cin).getTime()
  const end = todayRecord.value?.clockout_time
    ? new Date(todayRecord.value.clockout_time).getTime()
    : Date.now()

  return Math.max(0, end - start)
})

const workedHHMMSS = computed(() => {
  const ms = workedMs.value
  const totalSeconds = Math.floor(ms / 1000)
  const h = Math.floor(totalSeconds / 3600)
  const m = Math.floor((totalSeconds % 3600) / 60)
  const s = totalSeconds % 60
  return `${pad2(h)}:${pad2(m)}:${pad2(s)}`
})

function startTimerIfNeeded() {
  if (status.value !== 'IN') return
  stopTimer()
  intervalId = window.setInterval(() => {
    tick.value++
  }, 1000)
}

function stopTimer() {
  if (intervalId != null) {
    window.clearInterval(intervalId)
    intervalId = undefined
  }
}

async function loadTodayRecord() {
  const today = new Date().toISOString().slice(0, 10)
  const filter = `member="${currentUserId.value}" && date ~ "${today}"`

  const records = await pb.collection(Collections.Timesheet).getList(1, 1, { filter })
  todayRecord.value = records.items.length ? records.items[0] : null
}

async function compressFiles(files: File[]) {
  const options = { maxSizeMB: 1, maxWidthOrHeight: 1920, useWebWorker: true }
  return Promise.all(files.map(file => imageCompression(file, options)))
}

async function handleClockInPhotos(files: File[]) {
  clockInData.value.photos = await compressFiles(files)
}

async function handleClockOutPhotos(files: File[]) {
  clockOutData.value.photos = await compressFiles(files)
}

async function handleClockOutReceipts(files: File[]) {
  clockOutData.value.receipts = await compressFiles(files)
}

function setCurrentTime(mode: 'in' | 'out') {
  const now = new Date()
  const timeString = now.toLocaleTimeString('en-US', { hour: '2-digit', minute: '2-digit' })
  if (mode === 'in') clockInData.value.time = timeString
  else clockOutData.value.time = timeString
}

function setLocation(mode: 'in' | 'out') {
  if (!navigator.geolocation) return

  // eslint-disable-next-line @typescript-eslint/no-misused-promises
  navigator.geolocation.getCurrentPosition(async pos => {
    const lat = pos.coords.latitude
    const lon = pos.coords.longitude

    try {
      const res = await fetch(`https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lon}&format=json`)
      const data = await res.json()
      const address = data.display_name || `${lat}, ${lon}`

      if (mode === 'in') clockInData.value.location = address
      else clockOutData.value.location = address
    } catch {
      const fallback = `${lat}, ${lon}`
      if (mode === 'in') clockInData.value.location = fallback
      else clockOutData.value.location = fallback
    }
  })
}

function refreshInMeta() {
  setCurrentTime('in')
  setLocation('in')
}

function refreshOutMeta() {
  setCurrentTime('out')
  setLocation('out')
}

async function clockIn() {
  if (clockInData.value.photos.length < 3) {
    Notify.create({ type: 'negative', message: 'Upload at least 3 photos' })
    return
  }

  if (!clockInData.value.travel) {
    Notify.create({ type: 'negative', message: 'Select travel time' })
    return
  }

  submitting.value = true
  try {
    const formData = new FormData()

    formData.append('member', currentUserId.value)
    formData.append('date', new Date().toISOString().slice(0, 10))
    formData.append('clockin_time', new Date().toISOString()) // SaaS: server time is better; this is OK for now
    formData.append('clockin_location', clockInData.value.location)
    formData.append('clockin_travel', clockInData.value.travel)

    clockInData.value.photos.forEach(p => formData.append('clockin_photos', p))

    const record = await pb.collection(Collections.Timesheet).create(formData)

    todayRecord.value = record

    clockOutData.value.travel = clockInData.value.travel

    refreshOutMeta()
    startTimerIfNeeded()

    Notify.create({ type: 'positive', message: 'Clock in successful' })
  } catch (err: unknown) {
    Notify.create({ type: 'negative', message: errorMessage(err) })
  } finally {
    submitting.value = false
  }
}

async function clockOut() {
  if (!todayRecord.value) {
    Notify.create({ type: 'negative', message: 'No active clock-in record' })
    return
  }

  if (clockOutData.value.photos.length < 3) {
    Notify.create({ type: 'negative', message: 'Upload at least 3 photos' })
    return
  }

  submitting.value = true
  try {
    const formData = new FormData()

    formData.append('clockout_time', new Date().toISOString())
    formData.append('clockout_location', clockOutData.value.location)
    formData.append('clockout_travel', clockOutData.value.travel)
    formData.append('had_break', String(clockOutData.value.hadBreak))

    clockOutData.value.photos.forEach(p => formData.append('clockout_photos', p))
    clockOutData.value.receipts.forEach(r => formData.append('clockout_receipts', r))

    const record = await pb.collection(Collections.Timesheet).update(todayRecord.value.id, formData)
    todayRecord.value = record

    stopTimer()

    Notify.create({ type: 'positive', message: 'Clock out successful' })
  } catch (err: unknown) {
    Notify.create({ type: 'negative', message: errorMessage(err) })
  } finally {
    submitting.value = false
  }
}

function statusLabel(status?: string) {
  return STATUS_OPTIONS.find(o => o.value === status)?.label || status || '—'
}

function statusColor(status?: string) {
  switch (status) {
    case 'interested': return 'positive'
    case 'not_interested': return 'negative'
    case 'follow_up': return 'warning'
    case 'closed': return 'purple'
    default: return 'primary'
  }
}

function errorMessage(err: unknown, fallback = 'Unknown error'): string {
  return err instanceof Error && err.message ? err.message : fallback
}

async function loadReports() {
  const filter = `member="${currentUserId.value}" && date ~ "${reportDate.value}"`
  const records = await pb.collection(Collections.DailyReports).getFullList({
    filter,
    sort: '-created'
  })
  reports.value = records as unknown as DailyReportsRecord[]
}

function openReportDialog(report?: DailyReportsRecord) {
  reportDialog.id = report?.id || null
  Object.assign(reportForm, {
    institution: report?.institution || '',
    phone: report?.phone || '',
    email: report?.email || '',
    status: report?.status || 'visited',
    comments: report?.comments || '',
    location: report?.location || '',
    photos: []
  })
  reportPhotos.value = []
  reportDialog.open = true
}

async function handleReportPhotos(files: File[]) {
  reportForm.photos = await compressFiles(files)
}

async function saveReport() {
  if (!reportForm.institution.trim()) {
    Notify.create({ type: 'negative', message: 'Institution name is required' })
    return
  }

  reportSaving.value = true
  try {
    const formData = new FormData()

    formData.append('member', currentUserId.value)
    formData.append('date', reportDate.value)
    formData.append('institution', reportForm.institution.trim())
    formData.append('phone', reportForm.phone.trim())
    formData.append('email', reportForm.email.trim())
    formData.append('status', reportForm.status)
    formData.append('comments', reportForm.comments.trim())
    formData.append('location', reportForm.location.trim())

    reportForm.photos.forEach(p => formData.append('photos', p))

    if (reportDialog.id) {
      await pb.collection(Collections.DailyReports).update(reportDialog.id, formData)
      Notify.create({ type: 'positive', message: 'Visit updated' })
    } else {
      await pb.collection(Collections.DailyReports).create(formData)
      Notify.create({ type: 'positive', message: 'Visit added' })
    }

    reportDialog.open = false
    await loadReports()
  } catch (err: unknown) {
    Notify.create({ type: 'negative', message: errorMessage(err) })
  } finally {
    reportSaving.value = false
  }
}

async function deleteReport(report: DailyReportsRecord) {
  if (!window.confirm(`Delete visit to "${report.institution}"?`)) return

  try {
    await pb.collection(Collections.DailyReports).delete(report.id)
    Notify.create({ type: 'positive', message: 'Visit deleted' })
    await loadReports()
  } catch (err: unknown) {
    Notify.create({ type: 'negative', message: errorMessage(err) })
  }
}

/* =========================================================
   TABS — clock in/out vs daily report
========================================================= */

const tab = ref<'clock' | 'reports'>('clock')

const isManagement = computed(() => pb.authStore.record?.job !== 'Staff')

/* =========================================================
   MANAGEMENT — team-wide daily report view
========================================================= */

interface TeamReportRow {
  id: string
  staffName: string
  date: string
  institution: string
  phone: string
  email: string
  status: string
  comments: string
  photo_urls: string[]
}

const teamReports = ref<TeamReportRow[]>([])
const staffOptions = ref<{ label: string, value: string }[]>([])
const selectedStaff = ref<string | null>(null)
const monthView = ref('')
const teamLoading = ref(false)
const isMounted = ref(false)

const teamReportDialog = reactive<{ open: boolean, report: DailyReportsRecord | null }>({
  open: false,
  report: null
})

const teamColumns = [
  { name: 'date', label: 'Date', field: 'date', align: 'center' as const },
  { name: 'staff', label: 'Staff', field: 'staffName', align: 'center' as const },
  { name: 'institution', label: 'Institution', field: 'institution', align: 'center' as const },
  { name: 'phone', label: 'Phone', field: 'phone', align: 'center' as const },
  { name: 'email', label: 'Email', field: 'email', align: 'center' as const },
  { name: 'status', label: 'Status', field: 'status', align: 'center' as const },
  { name: 'comments', label: 'Comments', field: 'comments', align: 'left' as const },
  { name: 'photos', label: 'Photos', field: 'photo_urls', align: 'center' as const },
  { name: 'actions', label: 'Actions', field: 'id', align: 'center' as const }
]

function teamCountByStatus(status: string) {
  return teamReports.value.filter(r => r.status === status).length
}

function buildTeamFileUrl(collectionId: string, recordId: string, fileName: string) {
  return `${baseUrl}/api/files/${collectionId}/${recordId}/${fileName}`
}

function mapTeamReport(record: DailyReportsResponse<{ member: UsersResponse }>): TeamReportRow {
  const member = record.expand?.member

  return {
    id: record.id,
    staffName: member?.name || 'Unknown',
    date: record.date || '',
    institution: record.institution || '',
    phone: record.phone || '',
    email: record.email || '',
    status: record.status || 'visited',
    comments: record.comments || '',
    photo_urls: (record.photos || []).map((f: string) =>
      buildTeamFileUrl(record.collectionId, record.id, f)
    )
  }
}

async function fetchTeamStaff() {
  const staff = await pb.collection(Collections.Users).getFullList({
    filter: 'job != "Director"'
  })

  staffOptions.value = staff.map(user => ({
    label: user.name || 'Unknown',
    value: user.id
  }))
}

async function fetchTeamReports() {
  teamLoading.value = true

  try {
    const filters: string[] = []

    if (selectedStaff.value) {
      filters.push(`member="${selectedStaff.value}"`)
    }

    if (monthView.value) {
      const [year, month] = monthView.value.split('-')

      const start = `${year}-${month}-01`

      const nextMonth = new Date(Number(year), Number(month), 1)

      const end =
        `${nextMonth.getFullYear()}-${String(nextMonth.getMonth() + 1).padStart(2, '0')}-01`

      filters.push(`date >= "${start}" && date < "${end}"`)
    }

    const filter = filters.join(' && ')

    const records = await pb.collection(Collections.DailyReports).getFullList({
      expand: 'member',
      filter,
      sort: '-date,-created'
    })

    teamReports.value = records.map(mapTeamReport)
  } finally {
    teamLoading.value = false
  }
}

function openTeamCreateDialog() {
  teamReportDialog.report = null
  teamReportDialog.open = true
}

async function openTeamEditDialog(row: TeamReportRow) {
  try {
    const record = await pb.collection(Collections.DailyReports).getOne(row.id)
    teamReportDialog.report = record as unknown as DailyReportsRecord
    teamReportDialog.open = true
  } catch (err: unknown) {
    Notify.create({ type: 'negative', message: errorMessage(err, 'Unable to load report') })
  }
}

async function onTeamReportSaved() {
  await fetchTeamReports()
}

function exportTeamPDF() {
  const doc = new jsPDF()

  const rows = teamReports.value.map(r => [
    formatDate(r.date),
    r.staffName,
    r.institution,
    r.phone,
    r.email,
    statusLabel(r.status),
    r.comments
  ])

  autoTable(doc, {
    head: [['Date', 'Staff', 'Institution', 'Phone', 'Email', 'Status', 'Comments']],
    body: rows
  })

  doc.save('daily-reports.pdf')
}

onMounted(async () => {
  isMounted.value = true

  await loadTodayRecord()
  loading.value = false

  if (status.value === 'NONE') {
    refreshInMeta()
  } else if (status.value === 'IN') {
    clockOutData.value.travel = todayRecord.value?.clockin_travel || ''
    refreshOutMeta()
    startTimerIfNeeded()
  } else {
    stopTimer()
  }

  await loadReports()

  if (isManagement.value) {
    await fetchTeamStaff()
    await fetchTeamReports()
  }
})

onBeforeUnmount(() => {
  stopTimer()
})
</script>

<style scoped>
.reports-mobile-filters {
  margin: -4px 0 16px;
  padding: 10px 12px;
  border: 1px solid var(--q-color-grey-4);
  border-radius: 8px;
  background: white;
}

.reports-filter-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 16px;
}

/* Summary cards: keep all totals on a single line.
   4 cards need ~130px each, so drop to a single row from 600px up
   (the management view is only rendered for management users, and the
   moment there is not enough width per card they stack 2 / 1 per row). */
.reports-summary-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
}

@media (max-width: 599px) {
  .reports-summary-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 10px;
  }
}

.reports-summary-card {
  min-width: 0;
}

.reports-summary-card__body {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 12px 8px;
  text-align: center;
}

.reports-summary-card__label {
  font-size: 13px;
  line-height: 1.2;
  font-weight: 500;
  color: var(--q-color-grey-8);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 100%;
}

@media (max-width: 599px) {
  .reports-filter-grid {
    gap: 10px;
  }

  .reports-summary-card__body {
    padding: 10px 6px;
  }

  /* very narrow phones: allow the label to wrap instead of truncating */
  .reports-summary-card__label {
    white-space: normal;
    font-size: 12px;
  }
}
</style>