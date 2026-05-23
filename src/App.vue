<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

type Room = {
  id: number
  roomNo: string
  roomType: string
  rent: number | null
  deposit: number | null
  status: string
  tenantName: string | null
  tenantPhone: string | null
  contractStartDate: string | null
  contractEndDate: string | null
  note: string | null
}

type RoomPhoto = {
  id: number
  roomId: number
  photoUrl: string
  description: string | null
  createdAt: string
}

type ElectricityRecord = {
  id: number
  roomId: number
  recordMonth: string | null
  previousReading: number | null
  currentReading: number | null
  pricePerUnit: number | null
  usedUnits: number | null
  totalAmount: number | null
  note: string | null
}

const rooms = ref<Room[]>([])
const message = ref('')
const API_BASE = 'https://rental-api-4w5a.onrender.com'
const editingRoom = ref<Room | null>(null)

const roomPhotos = ref<RoomPhoto[]>([])
const showPhotoModal = ref(false)

const photoForm = ref({
  photoUrl: '',
  description: ''
})

const selectedRoom = ref<Room | null>(null)
const electricityRecords = ref<ElectricityRecord[]>([])
const showElectricityModal = ref(false) 

const electricityForm = ref({
  recordMonth: '',
  previousReading: null as number | null,
  currentReading: null as number | null,
  pricePerUnit: 5,
  note: ''
})

const maxUsedUnits = computed(() => {
  if (electricityRecords.value.length === 0) return 1

  const max = Math.max(
    ...electricityRecords.value.map(record => record.usedUnits ?? 0)
  )

  return max === 0 ? 1 : max
})

const electricityAmountPreview = computed(() => {
  const previous = electricityForm.value.previousReading
  const current = electricityForm.value.currentReading
  const price = electricityForm.value.pricePerUnit

  if (previous == null || current == null || price == null) {
    return 0
  }

  const usedUnits = current - previous
  return usedUnits > 0 ? usedUnits * price : 0
})

const totalReceivablePreview = computed(() => {
  const rent = selectedRoom.value?.rent ?? 0
  return rent + electricityAmountPreview.value
})

function startEdit(room: Room) {
  editingRoom.value = { ...room }
}

function cancelEdit() {
  editingRoom.value = null
}

async function saveRoom() {
  if (!editingRoom.value) return

  try {
    const response = await fetch(`${API_BASE}/api/rooms/${editingRoom.value.id}`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(editingRoom.value)
    })

    if (!response.ok) {
      throw new Error('修改失敗')
    }

    message.value = '房間資料已更新'
    editingRoom.value = null
    await loadRooms()
    } catch (error) {
    console.error(error)
    alert('新增電費紀錄失敗，請看 Render Logs')
    message.value = '新增電費紀錄失敗'
  }
}

async function loadElectricityRecords(room: Room) {
  selectedRoom.value = room
  showElectricityModal.value = true

  try {
    const response = await fetch(`${API_BASE}/api/electricity-records/room/${room.id}`)
    electricityRecords.value = await response.json()

    // 依月份排序，找最新一筆電費紀錄
    const sortedRecords = [...electricityRecords.value].sort((a, b) => {
      return String(b.recordMonth || '').localeCompare(String(a.recordMonth || ''))
    })

    const latestRecord = sortedRecords[0]

    // 如果有上一筆紀錄，就自動把上一筆「本期度數」帶入這次「上期度數」
    if (latestRecord && latestRecord.currentReading != null) {
      electricityForm.value.previousReading = latestRecord.currentReading
    } else {
      electricityForm.value.previousReading = null
    }
  } catch (error) {
    console.error(error)
    message.value = '讀取電費紀錄失敗'
  }
}

async function openPhotoModal(room: Room) {
  selectedRoom.value = room
  showPhotoModal.value = true
  photoForm.value = {
    photoUrl: '',
    description: ''
  }
  await loadRoomPhotos(room.id)
}

async function loadRoomPhotos(roomId: number) {
  try {
    const response = await fetch(`${API_BASE}/api/rooms/${roomId}/photos`)
    roomPhotos.value = await response.json()
  } catch (error) {
    console.error(error)
    alert('讀取房間照片失敗')
  }
}

async function createRoomPhoto() {
  if (!selectedRoom.value) return

  if (!photoForm.value.photoUrl) {
    alert('請輸入照片網址')
    return
  }

  try {
    const response = await fetch(`${API_BASE}/api/rooms/${selectedRoom.value.id}/photos`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(photoForm.value)
    })

    if (!response.ok) {
      throw new Error('新增照片失敗')
    }

    photoForm.value = {
      photoUrl: '',
      description: ''
    }

    await loadRoomPhotos(selectedRoom.value.id)
  } catch (error) {
    console.error(error)
    alert('新增照片失敗')
  }
}

async function deleteRoomPhoto(photoId: number) {
  if (!selectedRoom.value) return

  const confirmed = confirm('確定要刪除這張照片嗎？')
  if (!confirmed) return

  try {
    const response = await fetch(`${API_BASE}/api/room-photos/${photoId}`, {
      method: 'DELETE'
    })

    if (!response.ok) {
      throw new Error('刪除照片失敗')
    }

    await loadRoomPhotos(selectedRoom.value.id)
  } catch (error) {
    console.error(error)
    alert('刪除照片失敗')
  }
}

function closePhotoModal() {
  showPhotoModal.value = false
  selectedRoom.value = null
  roomPhotos.value = []
}

async function createElectricityRecord() {
  if (!selectedRoom.value) return

  try {
    
    const response = await fetch(`${API_BASE}/api/electricity-records/room/${selectedRoom.value.id}`, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify(electricityForm.value)
})

    if (!response.ok) {
      throw new Error('新增電費失敗')
    }

    electricityForm.value = {
      recordMonth: '',
      previousReading: null,
      currentReading: null,
      pricePerUnit: 5,
      note: ''
    }

    alert('新增電費紀錄成功')
    await loadElectricityRecords(selectedRoom.value)
  } catch (error) {
    console.error(error)
    message.value = '新增電費紀錄失敗'
  }
}

async function deleteElectricityRecord(id: number) {
  if (!confirm('確定要刪除這筆電費紀錄嗎？')) return

  try {
    const response = await fetch(`${API_BASE}/api/electricity-records/${id}`, {
      method: 'DELETE'
    })

    if (!response.ok) {
      throw new Error('刪除電費失敗')
    }

    if (selectedRoom.value) {
      await loadElectricityRecords(selectedRoom.value)
    }
  } catch (error) {
    console.error(error)
    message.value = '刪除電費紀錄失敗'
  }
}

function closeElectricityModal() {
  showElectricityModal.value = false
  selectedRoom.value = null
  electricityRecords.value = []
}

async function loadRooms() {
  try {
    const response = await fetch(`${API_BASE}/api/rooms`)
    rooms.value = await response.json()
  } catch (error) {
    console.error(error)
    message.value = '讀取房間資料失敗'
  }
}

onMounted(() => {
  loadRooms()
})
</script>

<template>
  <div class="page">
    <h1>租房管理系統</h1>
    <p class="subtitle">房間列表</p>

    <p v-if="message" class="message">{{ message }}</p>

    <div class="room-grid">
      <div v-for="room in rooms" :key="room.id" class="room-card">
        <div class="room-header">
          <h2>{{ room.roomNo }}</h2>
          <span :class="['status', room.status === '已入住' ? 'occupied' : 'empty']">
            {{ room.status }}
          </span>
        </div>

        <p>房型：{{ room.roomType }}</p>
        <p>租金：{{ room.rent ?? '未設定' }}</p>
        <p>押金：{{ room.deposit ?? '未設定' }}</p>
        <p>房客：{{ room.tenantName || '未入住' }}</p>
        <p>電話：{{ room.tenantPhone || '無' }}</p>
        <p>合約：{{ room.contractStartDate || '未設定' }} ～ {{ room.contractEndDate || '未設定' }}</p>
        <p>備註：{{ room.note || '無' }}</p>
        <button class="edit-btn" @click="startEdit(room)">編輯</button>
        <button class="electric-btn" @click="loadElectricityRecords(room)">電費紀錄</button>
        <button class="photo-btn" @click="openPhotoModal(room)">房間照片</button>
      </div>
    </div>
  </div>
  <div v-if="editingRoom" class="modal">
  <div class="modal-card">
    <h2>編輯房間 {{ editingRoom.roomNo }}</h2>

    <label>房號</label>
    <input v-model="editingRoom.roomNo" />

    <label>房型</label>
    <select v-model="editingRoom.roomType">
      <option value="套房">套房</option>
      <option value="雅房">雅房</option>
    </select>

    <label>租金</label>
    <input v-model.number="editingRoom.rent" type="number" />

    <label>押金</label>
    <input v-model.number="editingRoom.deposit" type="number" />

    <label>狀態</label>
    <select v-model="editingRoom.status">
      <option value="空房">空房</option>
      <option value="已入住">已入住</option>
      <option value="維修中">維修中</option>
    </select>

    <label>房客姓名</label>
    <input v-model="editingRoom.tenantName" />

    <label>房客電話</label>
    <input v-model="editingRoom.tenantPhone" />

    <label>合約開始日</label>
    <input v-model="editingRoom.contractStartDate" type="date" />

    <label>合約結束日</label>
    <input v-model="editingRoom.contractEndDate" type="date" />

    <label>備註</label>
    <textarea v-model="editingRoom.note"></textarea>

    <div class="modal-actions">
      <button class="save-btn" @click="saveRoom">儲存</button>
      <button class="cancel-btn" @click="cancelEdit">取消</button>
    </div>
  </div>
</div>

<div v-if="showPhotoModal" class="modal">
  <div class="modal-card">
    <h2>房間照片 {{ selectedRoom?.roomNo }}</h2>

    <div class="electric-form">
      <label>照片網址</label>
      <input
        v-model="photoForm.photoUrl"
        placeholder="請貼上照片網址"
      />

      <label>照片說明</label>
      <input
        v-model="photoForm.description"
        placeholder="例如：未入住照片、浴室、床位"
      />

      <button class="save-btn" @click="createRoomPhoto">新增照片</button>
    </div>

    <div v-if="roomPhotos.length === 0">
      <p>目前沒有房間照片</p>
    </div>

    <div v-if="roomPhotos.length > 0" class="photo-list">
      <div
        v-for="photo in roomPhotos"
        :key="photo.id"
        class="photo-item"
      >
        <img
          :src="photo.photoUrl"
          class="room-photo"
          alt="房間照片"
        />

        <p>{{ photo.description || '無說明' }}</p>

        <button
          class="delete-btn"
          @click="deleteRoomPhoto(photo.id)"
        >
          刪除
        </button>
      </div>
    </div>

    <div class="modal-actions">
      <button class="cancel-btn" @click="closePhotoModal">關閉</button>
    </div>
  </div>
</div>


<div v-if="showElectricityModal" class="modal">
  <div class="modal-card">
    <h2>{{ selectedRoom?.roomNo }} 房電費紀錄</h2>
     <div class="electric-form">
  <label>月份</label>
  <input v-model="electricityForm.recordMonth" type="date" />

  <label>上期度數</label>
  <input v-model.number="electricityForm.previousReading" type="number" />
<label>本期度數</label>
<input v-model.number="electricityForm.currentReading" type="number" />

<div class="money-row">
  <div class="money-field price-field">
    <label>單價</label>
    <input v-model.number="electricityForm.pricePerUnit" type="number" />
  </div>

  <div class="money-field amount-field">
    <label>電費</label>
    <div class="amount-box">
      {{ electricityAmountPreview }} 元
    </div>
  </div>
</div>

<label>總金額（房租加電費）</label>
<div class="amount-box total-box">
  {{ totalReceivablePreview }} 元
</div>

<button class="save-btn" @click="createElectricityRecord">新增電費紀錄</button>
</div>
    <div v-if="electricityRecords.length === 0">
      <p>目前沒有電費紀錄</p>
    </div>

    <div v-if="electricityRecords.length > 0" class="trend-box">
  <h3>用電趨勢圖</h3>

  <div
    v-for="record in electricityRecords"
    :key="'trend-' + record.id"
    class="trend-row"
  >
    <div class="trend-month">
      {{ record.recordMonth }}
    </div>

    <div class="trend-bar-wrap">
      <div
        class="trend-bar"
        :style="{ width: ((record.usedUnits ?? 0) / maxUsedUnits * 100) + '%' }"
      ></div>
    </div>

    <div class="trend-value">
      {{ record.usedUnits ?? 0 }} 度
    </div>
  </div>
</div>

    <div v-for="record in electricityRecords" :key="record.id" class="electric-record">
      <p>月份：{{ record.recordMonth }}</p>
      <p>上期度數：{{ record.previousReading }}</p>
      <p>本期度數：{{ record.currentReading }}</p>
      <p>使用度數：{{ record.usedUnits }}</p>
      <p>每度電費：{{ record.pricePerUnit }}</p>
      <p>應繳電費：{{ record.totalAmount }}</p>
      <p>備註：{{ record.note || '無' }}</p>
      <button class="delete-btn" @click="deleteElectricityRecord(record.id)">刪除</button>
    </div>

    <div class="modal-actions">
      <button class="cancel-btn" @click="closeElectricityModal">關閉</button>
    </div>
  </div>
</div>

</template>

<style scoped>
.page {
  min-height: 100vh;
  padding: 24px;
  background: #eef6ff;
  font-family: Arial, "Microsoft JhengHei", sans-serif;
}

h1 {
  margin: 0;
  text-align: center;
  color: #1f3b57;
}

.subtitle {
  text-align: center;
  color: #4f6b85;
  margin-bottom: 24px;
}

.message {
  text-align: center;
  color: red;
}

.room-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
  gap: 16px;
  max-width: 1100px;
  margin: 0 auto;
}

.room-card {
  background: #fff7ed;
  border-radius: 16px;
  padding: 18px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
  text-align: left;
}

.room-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 12px;
}

.room-header h2 {
  margin: 0;
  color: #1f3b57;
}

.status {
  padding: 6px 10px;
  border-radius: 999px;
  font-size: 14px;
  font-weight: bold;
}

.empty {
  background: #d1fae5;
  color: #047857;
}

.occupied {
  background: #fee2e2;
  color: #b91c1c;
}

p {
  margin: 8px 0;
  color: #333;
  text-align: left;
}

.edit-btn {
  margin-top: 12px;
  width: 100%;
  padding: 10px;
  border: none;
  border-radius: 10px;
  background: #2563eb;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

.modal {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 16px;
}

.modal-card {
  width: 100%;
  max-width: 420px;
  max-height: 90vh;
  overflow-y: auto;
  background: white;
  border-radius: 16px;
  padding: 20px;
}

.modal-card h2 {
  text-align: center;
  color: #1f3b57;
}

.modal-card label {
  display: block;
  margin-top: 10px;
  margin-bottom: 4px;
  color: #333;
}

.modal-card input,
.modal-card select,
.modal-card textarea {
  width: 100%;
  box-sizing: border-box;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 16px;
}

.modal-card textarea {
  min-height: 70px;
}

.modal-actions {
  display: flex;
  gap: 10px;
  margin-top: 16px;
}

.save-btn,
.cancel-btn {
  flex: 1;
  padding: 10px;
  border: none;
  border-radius: 10px;
  font-size: 16px;
  cursor: pointer;
}

.save-btn {
  background: #16a34a;
  color: white;
}

.cancel-btn {
  background: #e5e7eb;
  color: #333;
}

.electric-btn {
  margin-top: 10px;
  width: 100%;
  padding: 10px;
  border: none;
  border-radius: 10px;
  background: #f59e0b;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

.electric-record {
  background: #f8fafc;
  border-radius: 10px;
  padding: 10px;
  margin-top: 12px;
}

.electric-form {
  background: #fef3c7;
  border-radius: 12px;
  padding: 12px;
  margin-bottom: 16px;
}

.electric-form label {
  display: block;
  margin-top: 8px;
  margin-bottom: 4px;
  color: #333;
}

.electric-form input,
.electric-form textarea {
  width: 100%;
  box-sizing: border-box;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 8px;
  font-size: 16px;
}

.electric-form textarea {
  min-height: 60px;
}

@media (max-width: 600px) {
  .page {
    padding: 12px;
  }

  h1 {
    font-size: 36px;
  }

  .subtitle {
    font-size: 20px;
    margin-bottom: 18px;
  }

  .room-grid {
    grid-template-columns: 1fr;
    gap: 14px;
  }

  .room-card {
    padding: 18px;
    border-radius: 18px;
  }

  .room-header h2 {
    font-size: 30px;
  }

  .status {
    font-size: 18px;
    padding: 8px 14px;
  }

  p {
    font-size: 20px;
    line-height: 1.6;
  }

  .edit-btn,
  .electric-btn {
    font-size: 20px;
    padding: 14px;
    border-radius: 14px;
  }

  .modal {
    align-items: flex-start;
    padding: 12px;
    overflow-y: auto;
  }

  .modal-card {
    max-width: 100%;
    width: 100%;
    margin-top: 20px;
    padding: 18px;
    border-radius: 18px;
  }

  .modal-card h2 {
    font-size: 26px;
  }

  .modal-card label,
  .electric-form label {
    font-size: 18px;
  }

  .modal-card input,
  .modal-card select,
  .modal-card textarea,
  .electric-form input,
  .electric-form textarea {
    font-size: 18px;
    padding: 12px;
  }

  .save-btn,
  .cancel-btn {
    font-size: 20px;
    padding: 14px;
    border-radius: 14px;
  }

  .modal-actions {
    flex-direction: column;
  }

  .trend-row {
  grid-template-columns: 1fr;
  gap: 4px;
}

.trend-month,
.trend-value {
  font-size: 16px;
  text-align: left;
}

.trend-bar-wrap {
  height: 22px;
}

}

.delete-btn {
  margin-top: 8px;
  width: 100%;
  padding: 10px;
  border: none;
  border-radius: 10px;
  background: #dc2626;
  color: white;
  font-size: 16px;
  cursor: pointer;
}

.trend-box {
  background: #eff6ff;
  border-radius: 12px;
  padding: 12px;
  margin-bottom: 16px;
}

.trend-box h3 {
  margin: 0 0 12px 0;
  color: #1f3b57;
  text-align: center;
}

.trend-row {
  display: grid;
  grid-template-columns: 90px 1fr 70px;
  align-items: center;
  gap: 8px;
  margin-bottom: 10px;
}

.trend-month {
  font-size: 14px;
  color: #333;
}

.trend-bar-wrap {
  height: 18px;
  background: #dbeafe;
  border-radius: 999px;
  overflow: hidden;
}

.trend-bar {
  height: 100%;
  background: #2563eb;
  border-radius: 999px;
}

.trend-value {
  font-size: 14px;
  text-align: right;
  color: #333;
}

.photo-btn {
  width: 100%;
  margin-top: 10px;
  padding: 14px;
  border: none;
  border-radius: 16px;
  background: #22c55e;
  color: white;
  font-size: 20px;
  font-weight: bold;
  cursor: pointer;
}

.photo-btn:hover {
  background: #16a34a;
}

.photo-list {
  margin-top: 20px;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.photo-item {
  background: #fff7ed;
  border-radius: 16px;
  padding: 12px;
  border: 1px solid #fed7aa;
}

.room-photo {
  width: 100%;
  max-height: 260px;
  object-fit: cover;
  border-radius: 14px;
  margin-bottom: 10px;
}

.photo-item p {
  font-size: 18px;
  margin: 8px 0;
  color: #334155;
}

</style>