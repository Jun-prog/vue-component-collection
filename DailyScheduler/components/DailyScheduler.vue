<template>
  <div class="flex flex-col h-full p-4 md:p-8 max-w-4xl mx-auto">
    <h1 class="text-2xl font-bold mb-6">1日予定表</h1>
    
    <!-- 日付ナビゲーション -->
    <div class="flex items-center justify-between mb-4">
      <button 
        class="px-4 py-2 border rounded-md hover:bg-gray-100" 
        @click="handleDateChange(-1)"
      >
        前日
      </button>
      <h2 class="text-xl font-semibold">
        {{ formattedDate }}
      </h2>
      <button 
        class="px-4 py-2 border rounded-md hover:bg-gray-100" 
        @click="handleDateChange(1)"
      >
        翌日
      </button>
    </div>

    <!-- タスク入力エリア -->
    <div class="mb-4">
      <button 
        class="px-4 py-2 border rounded-md hover:bg-gray-100 mb-2" 
        @click="showTaskInput = !showTaskInput"
      >
        {{ showTaskInput ? "タスク入力を閉じる" : "タスクを箇条書きで入力" }}
      </button>
      
      <div v-if="showTaskInput" class="border rounded-lg p-4 bg-white">
        <label for="taskText" class="mb-2 block">
          タスクを1行ずつ入力してください（自動的にタイムラインに配置されます）
        </label>
        <textarea
          id="taskText"
          v-model="taskText"
          placeholder="例：
ミーティング資料の準備
メールの返信
プロジェクト計画の作成"
          class="w-full min-h-[150px] mb-2 p-2 border rounded-md"
        ></textarea>
        <div class="flex justify-end">
          <button 
            class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600"
            @click="handleAddTasksFromText"
          >
            タイムラインに追加
          </button>
        </div>
      </div>
    </div>

    <!-- タイムライン -->
    <div
      ref="timelineRef"
      class="relative border rounded-lg bg-white flex-1 min-h-[600px]"
      @click="handleTimelineClick"
    >
      <!-- 時間マーカー -->
      <div
        v-for="hour in hours"
        :key="hour"
        class="hour-marker absolute w-full border-t border-gray-200 flex items-center"
        :style="{ top: `${((hour - startHour) / (endHour - startHour + 1)) * 100}%` }"
      >
        <span class="text-xs text-gray-500 -mt-2 ml-2">{{ hour }}:00</span>
      </div>

      <!-- 予定ブロック -->
      <div
        v-for="schedule in schedules"
        :key="schedule.id"
        class="absolute left-[60px] right-4 rounded p-2 cursor-pointer text-white shadow-md transition-opacity hover:opacity-90"
        :class="schedule.color || 'bg-blue-500'"
        :style="{
          top: `${getSchedulePosition(schedule)}%`,
          height: `${getScheduleHeight(schedule)}%`,
        }"
        @click.stop="handleScheduleClick(schedule)"
        @mousedown.stop="handleDragStart(schedule.id, $event)"
      >
        <div class="font-medium truncate">{{ schedule.title }}</div>
        <div class="text-xs opacity-90">
          {{ schedule.startTime }} - {{ schedule.endTime }}
        </div>

        <!-- リサイズハンドル（上） -->
        <div 
          class="absolute top-0 left-0 right-0 h-2 cursor-ns-resize"
          @mousedown.stop="handleResizeStart(schedule.id, 'top', $event)"
        ></div>
        
        <!-- リサイズハンドル（下） -->
        <div 
          class="absolute bottom-0 left-0 right-0 h-2 cursor-ns-resize"
          @mousedown.stop="handleResizeStart(schedule.id, 'bottom', $event)"
        ></div>
      </div>

      <!-- 現在時刻のインジケーター -->
      <div 
        class="absolute left-0 right-0 border-t-2 border-red-500 z-10"
        :style="{ top: `${currentTimePosition}%` }"
      >
        <div class="w-3 h-3 rounded-full bg-red-500 -mt-1.5 -ml-1.5"></div>
      </div>
    </div>

    <!-- 予定追加ボタン -->
    <button
      class="fixed bottom-6 right-6 rounded-full w-12 h-12 shadow-lg bg-blue-500 text-white flex items-center justify-center hover:bg-blue-600"
      @click="handleAddButtonClick"
    >
      <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="h-6 w-6">
        <line x1="12" y1="5" x2="12" y2="19"></line>
        <line x1="5" y1="12" x2="19" y2="12"></line>
      </svg>
    </button>

    <!-- 予定編集モーダル -->
    <div v-if="isModalOpen" class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div class="bg-white rounded-lg p-6 w-full max-w-md">
        <div class="flex justify-between items-center mb-4">
          <h3 class="text-lg font-semibold">{{ editingSchedule.id ? "予定を編集" : "新しい予定" }}</h3>
          <button @click="isModalOpen = false" class="text-gray-500 hover:text-gray-700">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="h-6 w-6">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>
        
        <div class="grid gap-4 py-4">
          <div class="grid gap-2">
            <label for="title" class="font-medium">タイトル</label>
            <input
              id="title"
              v-model="editingSchedule.title"
              placeholder="予定のタイトル"
              class="w-full p-2 border rounded-md"
            />
          </div>
          
          <div class="grid grid-cols-2 gap-4">
            <div class="grid gap-2">
              <label for="startTime" class="font-medium">開始時間</label>
              <input
                id="startTime"
                type="time"
                v-model="editingSchedule.startTime"
                class="w-full p-2 border rounded-md"
              />
            </div>
            
            <div class="grid gap-2">
              <label for="endTime" class="font-medium">終了時間</label>
              <input
                id="endTime"
                type="time"
                v-model="editingSchedule.endTime"
                class="w-full p-2 border rounded-md"
              />
            </div>
          </div>
        </div>
        
        <div class="flex justify-between">
          <button 
            v-if="editingSchedule.id" 
            @click="handleDeleteSchedule" 
            class="px-4 py-2 bg-red-500 text-white rounded-md hover:bg-red-600 flex items-center"
          >
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="mr-2 h-4 w-4">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
            削除
          </button>
          <div v-else></div>
          
          <div class="flex gap-2">
            <button 
              @click="isModalOpen = false" 
              class="px-4 py-2 border rounded-md hover:bg-gray-100"
            >
              キャンセル
            </button>
            <button 
              @click="handleSaveSchedule" 
              class="px-4 py-2 bg-blue-500 text-white rounded-md hover:bg-blue-600 flex items-center"
            >
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="mr-2 h-4 w-4">
                <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
                <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
              </svg>
              保存
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

// 時間を分に変換する関数
const timeToMinutes = (time) => {
  const [hours, minutes] = time.split(":").map(Number)
  return hours * 60 + minutes
}

// 分を時間形式に変換する関数
const minutesToTime = (minutes) => {
  const hours = Math.floor(minutes / 60)
  const mins = minutes % 60
  return `${hours.toString().padStart(2, "0")}:${mins.toString().padStart(2, "0")}`
}

// 状態の定義
const date = ref(new Date())
const schedules = ref([
  {
    id: "1",
    title: "朝のミーティング",
    startTime: "09:00",
    endTime: "10:00",
    color: "bg-blue-500",
  },
  {
    id: "2",
    title: "プロジェクト作業",
    startTime: "11:00",
    endTime: "13:00",
    color: "bg-green-500",
  },
  {
    id: "3",
    title: "昼食",
    startTime: "13:00",
    endTime: "14:00",
    color: "bg-yellow-500",
  },
])

// 表示する時間範囲（6時から22時）
const startHour = 6
const endHour = 22
const hours = Array.from({ length: endHour - startHour + 1 }, (_, i) => startHour + i)

// ドラッグ関連の状態
const dragging = ref(null)
const resizing = ref(null)
const dragStartY = ref(0)
const initialPosition = ref(0)
const timelineRef = ref(null)

// モーダル関連の状態
const isModalOpen = ref(false)
const editingSchedule = ref({
  id: "",
  title: "",
  startTime: "09:00",
  endTime: "10:00",
})
const newScheduleTime = ref(null)

// タスク入力関連
const taskText = ref("")
const showTaskInput = ref(false)

// 日付のフォーマット
const formattedDate = computed(() => {
  const year = date.value.getFullYear()
  const month = date.value.getMonth() + 1
  const day = date.value.getDate()
  const weekdays = ['日', '月', '火', '水', '木', '金', '土']
  const weekday = weekdays[date.value.getDay()]
  return `${year}年${month}月${day}日（${weekday}）`
})

// 現在時刻の位置
const currentTimePosition = computed(() => {
  const now = new Date()
  const currentHour = now.getHours()
  const currentMinute = now.getMinutes()
  return ((currentHour + currentMinute / 60 - startHour) / (endHour - startHour + 1)) * 100
})

// スケジュールの位置を計算
const getSchedulePosition = (schedule) => {
  const startMinutes = timeToMinutes(schedule.startTime)
  return ((startMinutes / 60 - startHour) / (endHour - startHour + 1)) * 100
}

// スケジュールの高さを計算
const getScheduleHeight = (schedule) => {
  const startMinutes = timeToMinutes(schedule.startTime)
  const endMinutes = timeToMinutes(schedule.endTime)
  const duration = (endMinutes - startMinutes) / 60
  return (duration / (endHour - startHour + 1)) * 100
}

// テキストからタスクを解析してスケジュールに追加する関数
const handleAddTasksFromText = () => {
  if (!taskText.value.trim()) return
  
  // 各行をタスクとして解析
  const tasks = taskText.value
    .split('\n')
    .filter(line => line.trim())
    .map(line => line.trim())
  
  if (tasks.length === 0) return
  
  // 既存のスケジュールの時間帯を取得
  const occupiedTimeSlots = schedules.value.map(schedule => ({
    start: timeToMinutes(schedule.startTime),
    end: timeToMinutes(schedule.endTime)
  }))
  
  // 新しいスケジュールを作成
  const newSchedules = tasks.map(task => {
    // ランダムな時間帯を生成（30分単位）
    let startMinutes, endMinutes
    let attempts = 0
    let validTimeFound = false
    
    // 最大20回試行して重複しない時間帯を探す
    while (!validTimeFound && attempts < 20) {
      // 開始時間をランダムに選択（30分単位）
      const randomHour = startHour + Math.floor(Math.random() * (endHour - startHour))
      const randomMinute = Math.random() < 0.5 ? 0 : 30
      startMinutes = randomHour * 60 + randomMinute
      
      // タスクの長さは30分〜2時間でランダム
      const duration = [30, 60, 90, 120][Math.floor(Math.random() * 4)]
      endMinutes = startMinutes + duration
      
      // 終了時間が表示範囲を超えないように調整
      if (endMinutes > endHour * 60) {
        endMinutes = endHour * 60
      }
      
      // 既存のスケジュールと重複していないか確認
      validTimeFound = !occupiedTimeSlots.some(slot => 
        (startMinutes < slot.end && endMinutes > slot.start)
      )
      
      attempts++
    }
    
    // 重複する場合は、最後に追加する
    if (!validTimeFound) {
      // 最後のスケジュールの終了時間の後に配置
      const lastEndTime = occupiedTimeSlots.length > 0 
        ? Math.max(...occupiedTimeSlots.map(slot => slot.end))
        : startHour * 60
      
      startMinutes = Math.min(lastEndTime, (endHour - 1) * 60)
      endMinutes = Math.min(startMinutes + 60, endHour * 60) // デフォルトで1時間
    }
    
    // 新しい時間帯を占有リストに追加
    occupiedTimeSlots.push({
      start: startMinutes,
      end: endMinutes
    })
    
    // 色をランダムに選択
    const colors = ["bg-blue-500", "bg-green-500", "bg-yellow-500", "bg-purple-500", "bg-pink-500"]
    const randomColor = colors[Math.floor(Math.random() * colors.length)]
    
    return {
      id: Date.now().toString() + Math.random().toString(36).substr(2, 5),
      title: task,
      startTime: minutesToTime(startMinutes),
      endTime: minutesToTime(endMinutes),
      color: randomColor
    }
  })
  
  // スケジュールに追加
  schedules.value = [...schedules.value, ...newSchedules]
  
  // テキストエリアをクリア
  taskText.value = ""
  showTaskInput.value = false
}

// タイムラインのクリックイベント（新規予定の追加）
const handleTimelineClick = (e) => {
  if (e.target !== e.currentTarget && !e.target.classList.contains("hour-marker")) {
    return
  }

  const rect = timelineRef.value.getBoundingClientRect()
  if (!rect) return

  const y = e.clientY - rect.top
  const totalMinutes = (y / rect.height) * (endHour - startHour + 1) * 60
  const clickedTime = startHour * 60 + totalMinutes

  // 30分単位に丸める
  const roundedMinutes = Math.floor(clickedTime / 30) * 30
  const startTime = minutesToTime(roundedMinutes)
  const endTime = minutesToTime(roundedMinutes + 60) // デフォルトで1時間

  newScheduleTime.value = { startTime, endTime }
  editingSchedule.value = {
    id: "",
    title: "",
    startTime,
    endTime,
  }
  isModalOpen.value = true
}

// 予定ブロックのクリックイベント（編集）
const handleScheduleClick = (schedule) => {
  editingSchedule.value = { ...schedule }
  isModalOpen.value = true
}

// ドラッグ開始
const handleDragStart = (id, e) => {
  dragging.value = id
  dragStartY.value = e.clientY
  
  const schedule = schedules.value.find(s => s.id === id)
  if (schedule) {
    initialPosition.value = timeToMinutes(schedule.startTime)
  }
}

// リサイズ開始
const handleResizeStart = (id, edge, e) => {
  resizing.value = { id, edge }
  dragStartY.value = e.clientY
  
  const schedule = schedules.value.find(s => s.id === id)
  if (schedule) {
    if (edge === "top") {
      initialPosition.value = timeToMinutes(schedule.startTime)
    } else {
      initialPosition.value = timeToMinutes(schedule.endTime)
    }
  }
}

// マウス移動とマウスアップのイベントハンドラ
const handleMouseMove = (e) => {
  if (!dragging.value && !resizing.value) return
  if (!timelineRef.value) return

  const rect = timelineRef.value.getBoundingClientRect()
  const deltaY = e.clientY - dragStartY.value
  const deltaMinutes = (deltaY / rect.height) * (endHour - startHour + 1) * 60

  // 30分単位に丸める
  const roundedDeltaMinutes = Math.round(deltaMinutes / 30) * 30

  schedules.value = schedules.value.map(schedule => {
    if (dragging.value && schedule.id === dragging.value) {
      // ドラッグ中の場合は開始・終了時間を同時に移動
      const newStartMinutes = initialPosition.value + roundedDeltaMinutes
      const duration = timeToMinutes(schedule.endTime) - timeToMinutes(schedule.startTime)
      
      // 範囲外にならないように制限
      const limitedStartMinutes = Math.max(startHour * 60, Math.min(newStartMinutes, endHour * 60 - duration))
      
      return {
        ...schedule,
        startTime: minutesToTime(limitedStartMinutes),
        endTime: minutesToTime(limitedStartMinutes + duration)
      }
    } else if (resizing.value && schedule.id === resizing.value.id) {
      if (resizing.value.edge === "top") {
        // 上端のリサイズ（開始時間の変更）
        const newStartMinutes = initialPosition.value + roundedDeltaMinutes
        const limitedStartMinutes = Math.max(
          startHour * 60, 
          Math.min(newStartMinutes, timeToMinutes(schedule.endTime) - 30)
        )
        return {
          ...schedule,
          startTime: minutesToTime(limitedStartMinutes)
        }
      } else {
        // 下端のリサイズ（終了時間の変更）
        const newEndMinutes = initialPosition.value + roundedDeltaMinutes
        const limitedEndMinutes = Math.min(
          endHour * 60, 
          Math.max(newEndMinutes, timeToMinutes(schedule.startTime) + 30)
        )
        return {
          ...schedule,
          endTime: minutesToTime(limitedEndMinutes)
        }
      }
    }
    return schedule
  })
}

const handleMouseUp = () => {
  dragging.value = null
  resizing.value = null
}

// 予定の保存
const handleSaveSchedule = () => {
  if (!editingSchedule.value) return

  if (editingSchedule.value.id) {
    // 既存の予定を更新
    schedules.value = schedules.value.map(schedule => 
      schedule.id === editingSchedule.value.id ? editingSchedule.value : schedule
    )
  } else {
    // 新規予定を追加
    const newId = Date.now().toString()
    const colors = ["bg-blue-500", "bg-green-500", "bg-yellow-500", "bg-purple-500", "bg-pink-500"]
    const randomColor = colors[Math.floor(Math.random() * colors.length)]
    
    schedules.value.push({
      ...editingSchedule.value,
      id: newId,
      color: randomColor
    })
  }

  isModalOpen.value = false
  editingSchedule.value = {
    id: "",
    title: "",
    startTime: "09:00",
    endTime: "10:00",
  }
  newScheduleTime.value = null
}

// 予定の削除
const handleDeleteSchedule = () => {
  if (!editingSchedule.value.id) return
  
  schedules.value = schedules.value.filter(schedule => schedule.id !== editingSchedule.value.id)
  isModalOpen.value = false
  editingSchedule.value = {
    id: "",
    title: "",
    startTime: "09:00",
    endTime: "10:00",
  }
}

// 日付を変更
const handleDateChange = (increment) => {
  const newDate = new Date(date.value)
  newDate.setDate(newDate.getDate() + increment)
  date.value = newDate
}

// 予定追加ボタンのクリックイベント
const handleAddButtonClick = () => {
  const now = new Date()
  const currentHour = now.getHours()
  const roundedHour = Math.max(startHour, Math.min(currentHour, endHour))
  
  editingSchedule.value = {
    id: "",
    title: "",
    startTime: `${roundedHour.toString().padStart(2, "0")}:00`,
    endTime: `${(roundedHour + 1).toString().padStart(2, "0")}:00`,
  }
  isModalOpen.value = true
}

// イベントリスナーの設定と解除
onMounted(() => {
  window.addEventListener('mousemove', handleMouseMove)
  window.addEventListener('mouseup', handleMouseUp)
})

onUnmounted(() => {
  window.removeEventListener('mousemove', handleMouseMove)
  window.removeEventListener('mouseup', handleMouseUp)
})
</script>

<style scoped>
/* アニメーション効果 */
.absolute {
  transition: top 0.2s ease, height 0.2s ease;
}
</style>