<template>
  <div class="weekly-schedule">
    <!-- 화면에 보이는 UI -->
    <div class="screen-only">
      <!-- 표 타이틀 -->
      <h2 class="table-title">
        (
        <input
          type="date"
          v-model="selectedDate"
          @change="updateWeek"
          class="title-input"
        />
        ) 주간근무계획
        <button class="confirm-btn" @click="confirmWeeklySchedule">확정</button>
      </h2>
      <!-- 상단 입력 영역 -->
      <div class="input-section">
        <input
          v-model="newEmployeeName"
          type="text"
          placeholder="이름을 입력하세요"
        />
        <select v-model="newEmployeeType">
          <option value="정직원">정직원</option>
          <option value="파트타임">파트타임</option>
        </select>
        <button @click="addEmployee">추가</button>
        <button @click="openModal">관리</button>
        <button @click="openPrintModal">출력</button>
        <button @click="openHolidayManageModal">휴무관리</button>
        <button @click="openPlanManagementModal">계획 관리</button>
        <!-- 휴일 요청 버튼을 계획 관리 버튼 오른쪽에 위치 -->
        <button
          class="request-btn"
          :class="{ active: isRequestMode }"
          @click="toggleRequestMode"
          style="font-weight: bold; margin-left: 8px"
        >
          휴일 요청
        </button>
        <!-- '작성 일정' 버튼 추가 -->
        <button
          @click="openScheduleModal"
          style="font-weight: bold; margin-left: 8px"
        >
          작성 일정
        </button>
      </div>

      <!-- 직원명부 모달 -->
      <div v-if="showModal" class="modal-overlay" @click.self="closeModal">
        <div class="modal-content">
          <h3>직원명부</h3>
          <ul class="sortable-list">
            <li
              v-for="(emp, idx) in modalEmployees"
              :key="emp.name"
              :class="[
                'modal-emp',
                emp.type === '정직원' ? 'regular' : 'parttime',
                selectedIdx === idx ? 'selected' : '',
              ]"
              draggable="true"
              @dragstart="onModalDragStart(idx)"
              @dragover.prevent
              @drop="onModalDrop(idx)"
              @click="selectedIdx = idx"
            >
              {{ emp.name }} ({{ emp.type }})
              <button class="delete-btn" @click.stop="deleteModalEmployee(idx)">
                삭제
              </button>
            </li>
          </ul>
          <div class="modal-btns">
            <button @click="saveModalEmployees">저장</button>
            <button @click="closeModal">닫기</button>
          </div>
        </div>
      </div>

      <!-- 출력 옵션 모달 -->
      <div
        v-if="showPrintModal"
        class="modal-overlay"
        @click.self="closePrintModal"
      >
        <div class="modal-content">
          <h3>출력 옵션 선택</h3>
          <div class="modal-btns">
            <button @click="printSchedule">A4로 출력</button>
            <button @click="downloadPDF">PDF로 저장</button>
          </div>
          <div class="modal-btns" style="margin-top: 16px">
            <button @click="closePrintModal">닫기</button>
          </div>
        </div>
      </div>

      <!-- 메시지 박스(모달) -->
      <div v-if="showMessageBox" class="message-box-overlay">
        <div class="message-box-content">
          <div class="message-box-text">{{ messageBoxText }}</div>
          <div v-if="messageBoxType === 'confirm'">
            <button class="message-box-btn" @click="onMessageBoxConfirm">
              저장
            </button>
            <button
              class="message-box-btn"
              style="background: #888; margin-left: 12px"
              @click="onMessageBoxCancel"
            >
              취소
            </button>
          </div>
          <div v-else>
            <button class="message-box-btn" @click="showMessageBox = false">
              확인
            </button>
          </div>
        </div>
      </div>

      <!-- 휴무관리 모달 -->
      <div
        v-if="showHolidayManageModal"
        class="modal-overlay"
        @click.self="closeHolidayManageModal"
      >
        <div class="modal-content holiday-manage-modal">
          <h3>휴무관리</h3>
          <table class="holiday-manage-table">
            <thead>
              <tr>
                <th>직원명</th>
                <th>마지막 휴무 날짜</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="emp in employees" :key="emp.name">
                <td class="hm-emp-name">{{ emp.name }}</td>
                <td>
                  <input
                    type="date"
                    v-model="holidayManageMap[emp.name]"
                    class="hm-select"
                  />
                </td>
              </tr>
            </tbody>
          </table>
          <div class="modal-btns">
            <button @click="saveHolidayManageAll">저장</button>
            <button @click="closeHolidayManageModal">취소</button>
          </div>
        </div>
      </div>

      <!-- 계획 관리 모달 -->
      <div
        v-if="showPlanManagementModal"
        class="modal-overlay"
        @click.self="closePlanManagementModal"
      >
        <div class="modal-content">
          <h3>저장된 주간 계획</h3>
          <ul class="plan-list">
            <li
              v-for="(plan, weekKey) in sortedSavedPlans"
              :key="weekKey"
              class="plan-item"
            >
              <span>{{ formatWeekKey(weekKey) }}</span>
              <div class="plan-actions">
                <button @click="loadPlan(weekKey)">불러오기</button>
                <button @click="deletePlan(weekKey)" class="delete-btn">
                  삭제
                </button>
              </div>
            </li>
            <li v-if="Object.keys(savedPlans).length === 0">
              저장된 계획이 없습니다.
            </li>
          </ul>
          <div class="modal-btns">
            <button @click="closePlanManagementModal">닫기</button>
          </div>
        </div>
      </div>

      <div class="schedule-layout">
        <!-- 좌측 직원 구분란 -->
        <div class="employee-list">
          <div>
            <strong>정직원</strong>
            <div class="employee-blocks">
              <div
                v-for="(row, rowIdx) in regularEmployeeRows"
                :key="'reg-row-' + rowIdx"
                class="employee-row"
              >
                <div
                  v-for="emp in row"
                  :key="emp.name"
                  class="employee-block regular"
                  draggable="true"
                  @dragstart="onDragStart(emp)"
                >
                  {{ emp.name }}
                </div>
              </div>
            </div>
          </div>
          <div>
            <strong>파트타임</strong>
            <div class="employee-blocks">
              <div
                v-for="(row, rowIdx) in partTimeEmployeeRows"
                :key="'part-row-' + rowIdx"
                class="employee-row"
              >
                <div
                  v-for="emp in row"
                  :key="emp.name"
                  class="employee-block parttime"
                  draggable="true"
                  @dragstart="onDragStart(emp)"
                >
                  {{ emp.name }}
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 근무표 테이블 -->
        <div class="schedule-table-wrapper">
          <table class="schedule-table">
            <thead>
              <tr>
                <th>조</th>
                <th>성명</th>
                <th v-for="(day, index) in days" :key="day">
                  {{ day }}<br />
                  <span class="date-display">{{
                    getFormattedDate(index)
                  }}</span>
                </th>
              </tr>
            </thead>
            <tbody>
              <!-- A조 -->
              <tr v-for="(emp, idx) in aTeam" :key="'A' + idx">
                <td v-if="idx === 0" :rowspan="aTeam.length">A조</td>
                <td
                  class="drop-cell"
                  :class="
                    emp
                      ? emp.type === '정직원'
                        ? 'regular-bg'
                        : 'parttime-bg'
                      : ''
                  "
                  @click="onNameCellClick('A', idx)"
                  @dragover.prevent
                  @drop="onDrop('A', idx)"
                >
                  <span v-if="emp" @dblclick="onRemoveFromTeam('A', idx)">{{
                    emp.name
                  }}</span>
                </td>
                <td
                  v-for="(day, dayIdx) in days"
                  :key="'A' + idx + day"
                  @click="
                    isRequestMode
                      ? onRequestCellClick('A', idx, dayIdx)
                      : toggleCheckOrStar('A', idx, dayIdx)
                  "
                  @dblclick="toggleHoliday('A', idx, dayIdx)"
                  :class="[
                    emp && emp.holidays && emp.holidays[dayIdx]
                      ? 'holiday-cell'
                      : '',
                    requestedCells.has(`A-${idx}-${dayIdx}`)
                      ? 'requested-cell'
                      : '',
                  ]"
                >
                  <span
                    v-if="emp && emp.holidays && emp.holidays[dayIdx]"
                    class="holiday-text"
                    >휴일</span
                  >
                  <span
                    v-else-if="emp && emp.checked && emp.checked[dayIdx]"
                    class="checkmark"
                    >✔</span
                  >
                  <span
                    v-else-if="emp && emp.stars && emp.stars[dayIdx]"
                    class="star-mark"
                    >★</span
                  >
                </td>
              </tr>
              <!-- B조 -->
              <tr v-for="(emp, idx) in bTeam" :key="'B' + idx">
                <td v-if="idx === 0" :rowspan="bTeam.length">B조</td>
                <td
                  class="drop-cell"
                  :class="
                    emp
                      ? emp.type === '정직원'
                        ? 'regular-bg'
                        : 'parttime-bg'
                      : ''
                  "
                  @click="onNameCellClick('B', idx)"
                  @dragover.prevent
                  @drop="onDrop('B', idx)"
                >
                  <span v-if="emp" @dblclick="onRemoveFromTeam('B', idx)">{{
                    emp.name
                  }}</span>
                </td>
                <td
                  v-for="(day, dayIdx) in days"
                  :key="'B' + idx + day"
                  @click="
                    isRequestMode
                      ? onRequestCellClick('B', idx, dayIdx)
                      : toggleCheckOrStar('B', idx, dayIdx)
                  "
                  @dblclick="toggleHoliday('B', idx, dayIdx)"
                  :class="[
                    emp && emp.holidays && emp.holidays[dayIdx]
                      ? 'holiday-cell'
                      : '',
                    requestedCells.has(`B-${idx}-${dayIdx}`)
                      ? 'requested-cell'
                      : '',
                  ]"
                >
                  <span
                    v-if="emp && emp.holidays && emp.holidays[dayIdx]"
                    class="holiday-text"
                    >휴일</span
                  >
                  <span
                    v-else-if="emp && emp.checked && emp.checked[dayIdx]"
                    class="checkmark"
                    >✔</span
                  >
                  <span
                    v-else-if="emp && emp.stars && emp.stars[dayIdx]"
                    class="star-mark"
                    >★</span
                  >
                </td>
              </tr>
            </tbody>
          </table>
          <!-- 통계표 시작 -->
          <table class="schedule-table" style="margin-top: 32px">
            <thead>
              <tr>
                <th colspan="2">구분</th>
                <th v-for="day in days" :key="'stat-day-' + day">{{ day }}</th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <th rowspan="3">A조</th>
                <th>정직</th>
                <td v-for="i in 7" :key="'a-regular' + i">
                  {{ getStats('A', '정직원', i - 1) }}
                </td>
              </tr>
              <tr>
                <th>파트</th>
                <td v-for="i in 7" :key="'a-part' + i">
                  {{ getStats('A', '파트타임', i - 1) }}
                </td>
              </tr>
              <tr>
                <th>계</th>
                <td v-for="i in 7" :key="'a-sum' + i">
                  {{ getStats('A', '계', i - 1) }}
                </td>
              </tr>
              <tr>
                <th rowspan="3">B조</th>
                <th>정직</th>
                <td v-for="i in 7" :key="'b-regular' + i">
                  {{ getStats('B', '정직원', i - 1) }}
                </td>
              </tr>
              <tr>
                <th>파트</th>
                <td v-for="i in 7" :key="'b-part' + i">
                  {{ getStats('B', '파트타임', i - 1) }}
                </td>
              </tr>
              <tr>
                <th>계</th>
                <td v-for="i in 7" :key="'b-sum' + i">
                  {{ getStats('B', '계', i - 1) }}
                </td>
              </tr>
              <tr>
                <th colspan="2">총근무인원</th>
                <td v-for="i in 7" :key="'total' + i">
                  {{ getStats('total', '', i - 1) }}
                </td>
              </tr>
              <tr>
                <th colspan="2">휴무</th>
                <td v-for="i in 7" :key="'holiday' + i" class="holiday-text">
                  {{ getStats('holiday', '', i - 1) }}
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 출력 전용 UI -->
    <div class="print-only">
      <h2 class="table-title">{{ printTitleMonth }}월 근무일정표</h2>
      <table class="print-table">
        <thead>
          <tr>
            <th>근무</th>
            <th>구분</th>
            <th v-for="(day, index) in days" :key="'print-head-' + day">
              {{ day }}<br />
              <span class="date-display">{{ getFormattedDate(index) }}</span>
            </th>
          </tr>
        </thead>
        <tbody>
          <!-- A조 -->
          <tr>
            <th rowspan="2">A조</th>
            <th>직원</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-a-regular-' + dayIdx"
            >
              <div
                v-for="name in getWorkingEmployeesByType('A', '정직원', dayIdx)"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>파트</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-a-parttime-' + dayIdx"
            >
              <div
                v-for="name in getWorkingEmployeesByType(
                  'A',
                  '파트타임',
                  dayIdx
                )"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <!-- B조 -->
          <tr>
            <th rowspan="2">B조</th>
            <th>직원</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-b-regular-' + dayIdx"
            >
              <div
                v-for="name in getWorkingEmployeesByType('B', '정직원', dayIdx)"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>파트</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-b-parttime-' + dayIdx"
            >
              <div
                v-for="name in getWorkingEmployeesByType(
                  'B',
                  '파트타임',
                  dayIdx
                )"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th colspan="2">마감</th>
            <td v-for="(day, dayIdx) in days" :key="'print-closer-' + dayIdx">
              <div v-for="name in getClosers(dayIdx)" :key="name">
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th rowspan="2">휴무</th>
            <th>직원</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-holiday-regular-' + dayIdx"
            >
              <div
                v-for="name in getHolidayTakersByType('정직원', dayIdx)"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>파트</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-holiday-parttime-' + dayIdx"
            >
              <div
                v-for="name in getHolidayTakersByType('파트타임', dayIdx)"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th colspan="2">교육·지원</th>
            <td v-for="(day, dayIdx) in days" :key="'print-training-' + dayIdx">
              <div
                v-for="name in getTrainingSupportEmployees(dayIdx)"
                :key="name"
              >
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th colspan="2">내용</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-stats-' + dayIdx"
              class="stats-cell"
            >
              총원: {{ getDailyTotal(dayIdx) }}<br />
              근무: {{ getStats('total', '', dayIdx) }}<br />
              휴무: {{ getStats('holiday', '', dayIdx) }}<br />
              교/지:
              {{
                getTrainingSupportEmployees(dayIdx).length > 0
                  ? getTrainingSupportEmployees(dayIdx).length
                  : '-'
              }}<br />
              <strong>A조</strong><br />
              정직 : {{ getStats('A', '정직원', dayIdx) }}<br />
              파트 : {{ getStats('A', '파트타임', dayIdx) }}<br />
              <strong>B조</strong><br />
              정직 : {{ getStats('B', '정직원', dayIdx) }}<br />
              파트 : {{ getStats('B', '파트타임', dayIdx) }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- 작성 일정 모달 -->
    <div
      v-if="showScheduleModal"
      class="modal-overlay"
      @click.self="closeScheduleModal"
    >
      <div class="modal-content">
        <h3>작성 일정</h3>
        <table class="holiday-manage-table">
          <thead>
            <tr>
              <th>근무개시일</th>
              <th>계획표생성일</th>
              <th>등록마감일</th>
              <th>결재마감일</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in scheduleRows" :key="row.start">
              <td>{{ row.start }}</td>
              <td>{{ row.plan }}</td>
              <td>
                {{ row.reg }}
                <span class="reg-remain" style="color: #1976d2">{{
                  row.regRemain
                }}</span>
              </td>
              <td>{{ row.appr }}</td>
            </tr>
          </tbody>
        </table>
        <div class="modal-btns" style="margin-top: 16px">
          <button @click="closeScheduleModal">닫기</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
// 1. Imports
import { ref, computed, watch, onMounted } from 'vue';

// 2. State (상태 변수 선언)
const days = ['일', '월', '화', '수', '목', '금', '토'];
const selectedDate = ref(new Date().toISOString().slice(0, 10));
const newEmployeeName = ref('');
const newEmployeeType = ref('정직원');
const employees = ref([]);
const aTeam = ref(Array(7).fill(null));
const bTeam = ref(Array(7).fill(null));
const dragEmployee = ref(null);
const showModal = ref(false);
const showPrintModal = ref(false);
const modalEmployees = ref([]);
const selectedIdx = ref(null);
const showMessageBox = ref(false);
const messageBoxText = ref('');
const messageBoxType = ref('default');
const showHolidayManageModal = ref(false);
const holidayManageMap = ref({});
const showPlanManagementModal = ref(false);
const savedPlans = ref({});
let messageBoxCallback = null;
let dragIdx = null;
const isRequestMode = ref(false); // 휴일 요청 모드 on/off
const requestedCells = ref(new Set()); // 요청된 셀 key 집합
const showScheduleModal = ref(false); // 작성 일정 모달 표시 여부

const weekDates = computed(() => {
  const date = new Date(selectedDate.value);
  const day = date.getDay();
  const diff = date.getDate() - day;
  const sunday = new Date(date.setDate(diff));
  const dates = [];
  for (let i = 0; i < 7; i++) {
    const newDate = new Date(sunday);
    newDate.setDate(sunday.getDate() + i);
    dates.push(newDate);
  }
  return dates;
});

const unassignedEmployees = computed(() => {
  const assignedNames = [
    ...aTeam.value.filter((e) => e).map((e) => e.name),
    ...bTeam.value.filter((e) => e).map((e) => e.name),
  ];
  return employees.value.filter((e) => !assignedNames.includes(e.name));
});

const regularEmployees = computed(() =>
  unassignedEmployees.value.filter((e) => e.type === '정직원')
);

const partTimeEmployees = computed(() =>
  unassignedEmployees.value.filter((e) => e.type === '파트타임')
);

const regularEmployeeRows = computed(() =>
  chunkArray(regularEmployees.value, 2)
);
const partTimeEmployeeRows = computed(() =>
  chunkArray(partTimeEmployees.value, 2)
);

const printTitleMonth = computed(() => {
  if (weekDates.value.length > 3) {
    // A week can span two months. Use the month of the Wednesday (4th day)
    // as it's representative of the week's majority month.
    return weekDates.value[3].getMonth() + 1;
  }
  // Fallback to the selected date's month.
  return new Date(selectedDate.value).getMonth() + 1;
});

const sortedSavedPlans = computed(() => {
  const keys = Object.keys(savedPlans.value).sort().reverse();
  const sorted = {};
  for (const key of keys) {
    sorted[key] = savedPlans.value[key];
  }
  return sorted;
});

// 3. Business Logic (핵심 로직 함수)

function getFormattedDate(dayIndex) {
  const date = weekDates.value[dayIndex];
  return `${date.getMonth() + 1}/${date.getDate()}`;
}

function chunkArray(arr, size) {
  const result = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}

function addEmployee() {
  if (!newEmployeeName.value.trim()) {
    showMessageBoxWithText('이름을 입력하세요.');
    return;
  }
  if (employees.value.some((e) => e.name === newEmployeeName.value.trim())) {
    showMessageBoxWithText('이미 등록된 이름입니다.');
    return;
  }
  employees.value.push({
    name: newEmployeeName.value.trim(),
    type: newEmployeeType.value,
    lastHolidayDate: null,
  });
  newEmployeeName.value = '';
}

function onDragStart(emp) {
  dragEmployee.value = emp;
}

function sortTeamByEmployeeOrder(team) {
  const order = employees.value.map((e) => e.name);
  team.value = team.value
    .filter((e) => e)
    .sort((a, b) => {
      if (a.type !== b.type) return a.type === '정직원' ? -1 : 1;
      return order.indexOf(a.name) - order.indexOf(b.name);
    });
  while (team.value.length < 7) team.value.push(null);
}

function onDrop(teamType, idx) {
  if (!dragEmployee.value) return;
  if (
    aTeam.value.some((e) => e && e.name === dragEmployee.value.name) ||
    bTeam.value.some((e) => e && e.name === dragEmployee.value.name)
  )
    return;

  const masterEmp = employees.value.find(
    (e) => e.name === dragEmployee.value.name
  );
  const newEmp = {
    name: dragEmployee.value.name,
    type: dragEmployee.value.type,
    lastHolidayDate: masterEmp ? masterEmp.lastHolidayDate : null,
    checked: Array(7).fill(false),
    holidays: Array(7).fill(false),
    stars: Array(7).fill(false),
  };
  if (teamType === 'A') {
    aTeam.value[idx] = newEmp;
    sortTeamByEmployeeOrder(aTeam);
  } else {
    bTeam.value[idx] = newEmp;
    sortTeamByEmployeeOrder(bTeam);
  }
  dragEmployee.value = null;
}

function onRemoveFromTeam(teamType, idx) {
  if (teamType === 'A') {
    aTeam.value[idx] = null;
    sortTeamByEmployeeOrder(aTeam);
  } else {
    bTeam.value[idx] = null;
    sortTeamByEmployeeOrder(bTeam);
  }
}

function openModal() {
  modalEmployees.value = [...employees.value].sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
  selectedIdx.value = null;
  showModal.value = true;
}

function closeModal() {
  showModal.value = false;
}

function onModalDragStart(idx) {
  dragIdx = idx;
}

function onModalDrop(dropIdx) {
  if (dragIdx === null || dragIdx === dropIdx) return;
  const arr = modalEmployees.value;
  const moved = arr.splice(dragIdx, 1)[0];
  arr.splice(dropIdx, 0, moved);
  dragIdx = null;
  modalEmployees.value.sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
}

function deleteModalEmployee(idx) {
  modalEmployees.value.splice(idx, 1);
  selectedIdx.value = null;
  modalEmployees.value.sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
}

function saveModalEmployees() {
  employees.value = [...modalEmployees.value].sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
  sortTeamByEmployeeOrder(aTeam);
  sortTeamByEmployeeOrder(bTeam);
  closeModal();
}

function openPrintModal() {
  showPrintModal.value = true;
}

function closePrintModal() {
  showPrintModal.value = false;
}

function printSchedule() {
  window.print();
  closePrintModal();
}

function downloadPDF() {
  window.print();
  closePrintModal();
}

function showMessageBoxWithText(text, type = 'default', callback = null) {
  messageBoxText.value = text;
  showMessageBox.value = true;
  messageBoxType.value = type;
  messageBoxCallback = callback;
}

function onMessageBoxConfirm() {
  showMessageBox.value = false;
  if (messageBoxCallback) messageBoxCallback();
  messageBoxCallback = null;
}
function onMessageBoxCancel() {
  showMessageBox.value = false;
  messageBoxCallback = null;
}

function toggleHoliday(teamType, idx, dayIdx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp) return;

  if (!emp.holidays) emp.holidays = Array(7).fill(false);
  const isHoliday = emp.holidays[dayIdx];
  const currentCount = emp.holidays.filter(Boolean).length;
  const maxHoliday = emp.type === '정직원' ? 2 : 1;

  const newHolidayDate = weekDates.value[dayIdx];
  const masterEmp = employees.value.find((e) => e.name === emp.name);
  const lastHolidayDate = masterEmp ? masterEmp.lastHolidayDate : null;

  const getDaysDiff = (dateStr1, dateStr2) => {
    if (!dateStr1 || !dateStr2) return Infinity;
    const date1 = new Date(dateStr1);
    const date2 = new Date(dateStr2);
    date1.setHours(0, 0, 0, 0);
    date2.setHours(0, 0, 0, 0);
    return Math.round(Math.abs(date2 - date1) / (1000 * 60 * 60 * 24));
  };

  if (emp.type === '정직원') {
    const firstHolidayIdxInWeek = emp.holidays.findIndex((h) => h);

    if (firstHolidayIdxInWeek !== -1 && !isHoliday) {
      // 두 번째 휴무
      const firstHolidayDate = weekDates.value[firstHolidayIdxInWeek];
      const diff = getDaysDiff(
        firstHolidayDate.toISOString().slice(0, 10),
        newHolidayDate.toISOString().slice(0, 10)
      );

      if (diff > 6) {
        showMessageBoxWithText(
          '정직원의 두 휴일 사이 간격은 6일을 초과할 수 없습니다.'
        );
        return;
      }
      if (currentCount >= maxHoliday) {
        showMessageBoxWithText(
          '정직원은 휴일을 최대 2회만 지정할 수 있습니다.'
        );
        return;
      }
    } else if (!isHoliday) {
      // 첫 번째 휴무
      if (lastHolidayDate) {
        const diff = getDaysDiff(
          lastHolidayDate,
          newHolidayDate.toISOString().slice(0, 10)
        );
        if (diff > 6) {
          showMessageBoxWithText(
            `정직원은 이전 휴무일(${lastHolidayDate})로부터 6일 이내에 휴무를 지정해야 합니다.`
          );
          return;
        }
      }
      if (currentCount >= maxHoliday) {
        showMessageBoxWithText(
          '정직원은 휴일을 최대 2회만 지정할 수 있습니다.'
        );
        return;
      }
    } else if (isHoliday) {
      // 휴일 해제 시 규칙 위반 체크
      // 1. 해당 휴일을 해제
      emp.holidays[dayIdx] = false;
      // 2. 남아있는 휴일 인덱스 찾기
      const remainIdx = emp.holidays.findIndex((h) => h);
      if (remainIdx !== -1) {
        // 남아있는 휴일이 있으면, 이전 주 마지막 휴무일과의 차이 체크
        const remainDate = weekDates.value[remainIdx];
        if (lastHolidayDate) {
          const diff = getDaysDiff(
            lastHolidayDate,
            remainDate.toISOString().slice(0, 10)
          );
          if (diff > 6) {
            // 규칙 위반: 사용자에게 확인 요청
            showMessageBoxWithText(
              `앞의 휴일을 해제하면 남은 휴일이 이전 휴무일(${lastHolidayDate})로부터 6일 초과로 지정됩니다. 계속하시겠습니까?`,
              'confirm',
              () => {
                // 확인 시: 두 휴일 모두 해제
                emp.holidays[dayIdx] = false;
                emp.holidays[remainIdx] = false;
                showMessageBoxWithText(
                  '규칙 위반으로 두 휴일 모두 해제되었습니다.'
                );
              }
            );
            // 취소 시: 원래 휴일 상태로 되돌리기
            emp.holidays[dayIdx] = true;
            return;
          }
        }
      }
      return;
    }
  } else {
    // 파트타임: 1주일에 1번만 휴무 지정 가능
    if (!isHoliday) {
      if (currentCount >= 1) {
        showMessageBoxWithText(
          '파트타임은 한 주에 한 번만 휴일을 지정할 수 있습니다.'
        );
        return;
      }
      // 이전 휴무일로부터 6일 이내 규정은 삭제됨
    }
  }
  emp.holidays[dayIdx] = !isHoliday;
}

function toggleCheckOrStar(teamType, idx, dayIdx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp || (emp.holidays && emp.holidays[dayIdx])) return;

  if (!emp.checked) emp.checked = Array(7).fill(false);
  if (!emp.stars) emp.stars = Array(7).fill(false);

  if (teamType === 'B' && emp.type === '정직원') {
    if (emp.checked[dayIdx]) {
      emp.checked[dayIdx] = false;
      emp.stars[dayIdx] = true;
    } else if (emp.stars[dayIdx]) {
      emp.stars[dayIdx] = false;
    } else {
      emp.checked[dayIdx] = true;
    }
  } else {
    emp.checked[dayIdx] = !emp.checked[dayIdx];
    emp.stars[dayIdx] = false;
  }
}

function onNameCellClick(teamType, idx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp) return;

  if (!emp.checked) emp.checked = Array(7).fill(false);
  if (!emp.stars) emp.stars = Array(7).fill(false);
  if (!emp.holidays) emp.holidays = Array(7).fill(false);

  const checkedCells = emp.checked.filter(Boolean).length;
  if (checkedCells === 0) {
    for (let i = 0; i < 7; i++) {
      if (!emp.holidays[i] && !emp.stars[i]) {
        emp.checked[i] = true;
      }
    }
  } else {
    emp.checked = Array(7).fill(false);
  }
}

function getStats(teamType, empType, dayIdx) {
  let val = 0;
  const getTeamVal = (arr, type) =>
    arr.reduce((acc, emp) => {
      if (emp && !emp.holidays?.[dayIdx]) {
        const isWorking = emp.checked?.[dayIdx] || emp.stars?.[dayIdx];
        if (isWorking) {
          if (type === '계' || emp.type === type) {
            return acc + 1;
          }
        }
      }
      return acc;
    }, 0);

  if (teamType === 'A' || teamType === 'B') {
    const targetTeam = teamType === 'A' ? aTeam.value : bTeam.value;
    if (empType === '정직원' || empType === '파트타임' || empType === '계') {
      val = getTeamVal(targetTeam, empType);
    }
  } else if (teamType === 'total') {
    val = getTeamVal(aTeam.value, '계') + getTeamVal(bTeam.value, '계');
  } else if (teamType === 'holiday') {
    val = [...aTeam.value, ...bTeam.value].filter(
      (e) => e && e.holidays && e.holidays[dayIdx]
    ).length;
  }
  return val === 0 ? '-' : val;
}

const getWorkingEmployees = (team, dayIdx) => {
  const targetTeam = team === 'A' ? aTeam.value : bTeam.value;
  return targetTeam
    .filter(
      (emp) =>
        emp &&
        !emp.holidays?.[dayIdx] &&
        (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
    )
    .map((emp) => emp.name);
};

const getWorkingEmployeesByType = (team, empType, dayIdx) => {
  const targetTeam = team === 'A' ? aTeam.value : bTeam.value;
  return targetTeam
    .filter(
      (emp) =>
        emp &&
        emp.type === empType &&
        !emp.holidays?.[dayIdx] &&
        (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
    )
    .map((emp) => emp.name);
};

const getClosers = (dayIdx) => {
  return bTeam.value
    .filter(
      (emp) =>
        emp &&
        emp.type === '정직원' &&
        emp.stars?.[dayIdx] &&
        !emp.holidays?.[dayIdx]
    )
    .map((emp) => emp.name);
};

const getHolidayTakers = (dayIdx) => {
  return [...aTeam.value, ...bTeam.value]
    .filter((emp) => emp && emp.holidays?.[dayIdx])
    .map((emp) => emp.name);
};

const getHolidayTakersByType = (empType, dayIdx) => {
  return [...aTeam.value, ...bTeam.value]
    .filter((emp) => emp && emp.type === empType && emp.holidays?.[dayIdx])
    .map((emp) => emp.name);
};

const getTrainingSupportEmployees = (dayIdx) => {
  const allTeamMembers = [...aTeam.value, ...bTeam.value];
  return allTeamMembers
    .filter(
      (emp) =>
        emp &&
        !emp.holidays?.[dayIdx] &&
        !emp.checked?.[dayIdx] &&
        !emp.stars?.[dayIdx]
    )
    .map((emp) => emp.name);
};

const getDailyTotal = (dayIdx) => {
  const working = getStats('total', '', dayIdx);
  const holiday = getStats('holiday', '', dayIdx);
  const training = getTrainingSupportEmployees(dayIdx).length;
  const total =
    (working === '-' ? 0 : working) +
    (holiday === '-' ? 0 : holiday) +
    training;
  return total === 0 ? '-' : total;
};

function openHolidayManageModal() {
  holidayManageMap.value = {};
  employees.value.forEach((emp) => {
    holidayManageMap.value[emp.name] = emp.lastHolidayDate || '';
  });
  showHolidayManageModal.value = true;
}

function closeHolidayManageModal() {
  showHolidayManageModal.value = false;
}

function saveHolidayManageAll() {
  employees.value.forEach((emp) => {
    if (holidayManageMap.value.hasOwnProperty(emp.name)) {
      emp.lastHolidayDate = holidayManageMap.value[emp.name];
    }
  });
  showMessageBoxWithText('모든 직원의 마지막 휴무 날짜가 저장되었습니다.');
  closeHolidayManageModal();
}

function openPlanManagementModal() {
  showPlanManagementModal.value = true;
}
function closePlanManagementModal() {
  showPlanManagementModal.value = false;
}

function formatWeekKey(weekKey) {
  const date = new Date(weekKey);
  const endDate = new Date(date);
  endDate.setDate(date.getDate() + 6);
  return `${date.getMonth() + 1}/${date.getDate()} ~ ${
    endDate.getMonth() + 1
  }/${endDate.getDate()} 주간 계획`;
}

function loadPlan(weekKey) {
  selectedDate.value = weekKey;
  closePlanManagementModal();
}

function deletePlan(weekKey) {
  showMessageBoxWithText(
    `${formatWeekKey(weekKey)}을(를) 삭제하시겠습니까?`,
    'confirm',
    () => {
      delete savedPlans.value[weekKey];
      showMessageBoxWithText('삭제되었습니다.');
      if (getWeekKey(new Date(selectedDate.value)) === weekKey) {
        aTeam.value = Array(7).fill(null);
        bTeam.value = Array(7).fill(null);
      }
    }
  );
}

function getWeekKey(date) {
  const d = new Date(date);
  const day = d.getDay();
  const diff = d.getDate() - day;
  const sunday = new Date(d.setDate(diff));
  return sunday.toISOString().slice(0, 10);
}

function updateTeamsForSelectedDate(dateStr) {
  const weekKey = getWeekKey(new Date(dateStr));
  const plan = savedPlans.value[weekKey];
  if (plan) {
    aTeam.value = plan.aTeam;
    bTeam.value = plan.bTeam;
  } else {
    aTeam.value = Array(7).fill(null);
    bTeam.value = Array(7).fill(null);
  }
}

const STORAGE_KEY = 'daiso-weekly-plans-v1';

function saveToStorage() {
  const data = {
    employees: employees.value,
    savedPlans: savedPlans.value,
  };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}

function loadFromStorage() {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    try {
      const parsed = JSON.parse(data);
      if (
        parsed.employees &&
        parsed.employees.some((e) => e.hasOwnProperty('lastHolidayIndex'))
      ) {
        parsed.employees.forEach((emp) => {
          if (emp.hasOwnProperty('lastHolidayIndex')) {
            emp.lastHolidayDate = null;
            delete emp.lastHolidayIndex;
          }
        });
      }
      employees.value = parsed.employees || [];
      savedPlans.value = parsed.savedPlans || {};
    } catch (e) {
      console.error('Failed to load data from storage', e);
      employees.value = [];
      savedPlans.value = {};
    }
  }
}

function confirmWeeklySchedule() {
  showMessageBoxWithText(
    '현재 계획을 확정하고 저장하시겠습니까?',
    'confirm',
    () => {
      employees.value.forEach((masterEmp) => {
        const teamEmp =
          aTeam.value.find((e) => e && e.name === masterEmp.name) ||
          bTeam.value.find((e) => e && e.name === masterEmp.name);
        if (teamEmp && teamEmp.holidays) {
          let lastHolidayInWeekIdx = -1;
          for (let i = teamEmp.holidays.length - 1; i >= 0; i--) {
            if (teamEmp.holidays[i]) {
              lastHolidayInWeekIdx = i;
              break;
            }
          }
          if (lastHolidayInWeekIdx !== -1) {
            const lastDate = weekDates.value[lastHolidayInWeekIdx];
            masterEmp.lastHolidayDate = lastDate.toISOString().slice(0, 10);
          }
        }
      });

      const weekKey = getWeekKey(new Date(selectedDate.value));
      savedPlans.value[weekKey] = {
        aTeam: JSON.parse(JSON.stringify(aTeam.value)),
        bTeam: JSON.parse(JSON.stringify(bTeam.value)),
      };

      const planKeys = Object.keys(savedPlans.value).sort();
      if (planKeys.length > 4) {
        const oldestKey = planKeys[0];
        delete savedPlans.value[oldestKey];
      }

      showMessageBoxWithText('계획이 확정 및 저장되었습니다.');
    }
  );
}

// 4. Watchers & Lifecycle (감시 및 생명주기 훅)
watch(selectedDate, (newDateStr) => {
  updateTeamsForSelectedDate(newDateStr);
});

watch([employees, savedPlans], saveToStorage, {
  deep: true,
});

onMounted(() => {
  loadFromStorage();
  updateTeamsForSelectedDate(selectedDate.value);
});

// 2. 휴일 요청 버튼 토글 함수
function toggleRequestMode() {
  isRequestMode.value = !isRequestMode.value;
}

// 3. 셀 클릭 시 요청 토글 함수
function onRequestCellClick(teamType, idx, dayIdx) {
  if (isRequestMode.value) {
    const key = `${teamType}-${idx}-${dayIdx}`;
    if (requestedCells.value.has(key)) {
      requestedCells.value.delete(key);
    } else {
      requestedCells.value.add(key);
    }
  }
}

// 작성 일정 모달 열기 함수
function openScheduleModal() {
  showScheduleModal.value = true;
}
function closeScheduleModal() {
  showScheduleModal.value = false;
}

// 근무개시일(오늘 기준 미래 2주 단위 4개) 자동 생성, 결재마감일이 지난 행은 제외
const getFutureScheduleRows = () => {
  const result = [];
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  let base = new Date('2025-07-20');
  // base가 오늘보다 과거면, 오늘 이후의 가장 가까운 2주 단위 일요일로 맞춤
  while (base < today) {
    base.setDate(base.getDate() + 14);
  }
  let count = 0;
  let i = 0;
  while (count < 4) {
    const start = new Date(base);
    start.setDate(base.getDate() + i * 14);
    // 계획표생성일: 15일 전
    const planDate = new Date(start);
    planDate.setDate(start.getDate() - 15);
    // 등록마감일: 10일 전
    const regDate = new Date(start);
    regDate.setDate(start.getDate() - 10);
    // 결재마감일: 8일 전
    const apprDate = new Date(start);
    apprDate.setDate(start.getDate() - 8);
    // 결재마감일이 오늘보다 과거면 제외
    if (apprDate < today) {
      i++;
      continue;
    }
    // 등록마감일 (며칠전)
    const diff = Math.ceil((regDate - today) / (1000 * 60 * 60 * 24));
    let regRemain = '';
    if (diff > 0) regRemain = `(${diff}일전)`;
    else if (diff === 0) regRemain = '(오늘)';
    else regRemain = '(마감)';
    // 날짜 포맷 YYYY-MM-DD
    const fmt = (d) =>
      `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, '0')}-${String(
        d.getDate()
      ).padStart(2, '0')}`;
    result.push({
      start: fmt(start),
      plan: fmt(planDate),
      reg: fmt(regDate),
      regRemain,
      appr: fmt(apprDate),
    });
    count++;
    i++;
  }
  return result;
};
const scheduleRows = getFutureScheduleRows();
</script>

<style>
.print-only {
  display: none;
}
@media print {
  @page {
    size: A4 portrait;
    margin: 0mm 10mm 0mm 10mm;
  }
  html,
  body {
    height: 100%;
  }
  body {
    -webkit-print-color-adjust: exact !important;
    print-color-adjust: exact !important;
  }
  .screen-only {
    display: none;
  }
  .print-only {
    display: flex;
    flex-direction: column;
    height: 100%;
    page-break-inside: avoid;
    break-inside: avoid;
  }
  .table-title {
    flex-shrink: 0;
    text-align: center;
    font-size: 2rem;
    margin-bottom: 24px;
    font-weight: bold;
  }
  .print-table {
    width: 160mm;
    table-layout: fixed;
    border-collapse: collapse;
    font-size: 14px;
    flex-grow: 1;
    page-break-inside: avoid;
    break-inside: avoid;
  }
  .print-table th,
  .print-table td {
    border: 1px solid #000;
    width: 20mm;
    word-break: break-all;
    white-space: pre-line;
    padding: 0;
    text-align: center;
    vertical-align: middle;
  }
  .print-table thead tr:first-child th {
    height: 15mm;
  }
  /* A조 직원 */
  .print-table tbody tr:nth-child(1) th,
  .print-table tbody tr:nth-child(1) td {
    height: 30mm;
  }
  /* A조 파트 */
  .print-table tbody tr:nth-child(2) th,
  .print-table tbody tr:nth-child(2) td {
    height: 20mm;
  }
  /* B조 직원 */
  .print-table tbody tr:nth-child(3) th,
  .print-table tbody tr:nth-child(3) td {
    height: 30mm;
  }
  /* B조 파트 */
  .print-table tbody tr:nth-child(4) th,
  .print-table tbody tr:nth-child(4) td {
    height: 20mm;
  }
  /* 마감 */
  .print-table tbody tr:nth-child(5) th,
  .print-table tbody tr:nth-child(5) td {
    height: 25mm;
  }
  /* 휴무 직원 */
  .print-table tbody tr:nth-child(6) th,
  .print-table tbody tr:nth-child(6) td {
    height: 15mm;
  }
  /* 휴무 파트 */
  .print-table tbody tr:nth-child(7) th,
  .print-table tbody tr:nth-child(7) td {
    height: 15mm;
  }
  /* 교육·지원 */
  .print-table tbody tr:nth-child(8) th,
  .print-table tbody tr:nth-child(8) td {
    height: 15mm;
  }
  /* 내용 */
  .print-table tbody tr:nth-child(9) th,
  .print-table tbody tr:nth-child(9) td {
    height: 55mm;
    vertical-align: middle;
  }
  .blank-header {
    position: relative;
  }
}
.message-box-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.2);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
}
.message-box-content {
  background: #fff;
  border-radius: 12px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.18);
  padding: 32px 32px 24px 32px;
  min-width: 280px;
  display: flex;
  flex-direction: column;
  align-items: center;
}
.message-box-text {
  font-size: 1.1rem;
  margin-bottom: 24px;
  text-align: center;
  color: #222;
}
.message-box-btn {
  background: #1976d2;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 8px 28px;
  font-size: 1.1rem;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}
.message-box-btn:hover {
  background: #1251a3;
}
.confirm-btn {
  margin-left: 16px;
  padding: 6px 18px;
  font-size: 1.05rem;
  background: #1976d2;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}
.confirm-btn:hover {
  background: #1251a3;
}
.rewrite-btn {
  margin-left: 12px;
  padding: 6px 18px;
  font-size: 1.05rem;
  background: #ff9800;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}
.rewrite-btn:hover {
  background: #f57c00;
}
.request-btn.active {
  background: #ffeaea;
  color: #c62828;
  border: 2px solid #ffc1c1;
}
.requested-cell {
  background-color: #ffeaea !important;
  border: 2px solid #ffc1c1 !important;
}
</style>

<style scoped>
.weekly-schedule {
  font-family: '맑은 고딕', Arial, sans-serif;
  margin: 24px;
}
.input-section {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
}
.schedule-layout {
  display: flex;
  align-items: flex-start;
}
.employee-list {
  min-width: 120px;
  margin-right: 16px;
  border: 1px solid #333;
  padding: 8px;
  height: 400px;
  display: flex;
  flex-direction: column;
  justify-content: flex-start;
  gap: 24px;
}
.employee-list strong {
  display: block;
  margin-bottom: 4px;
}
.employee-blocks {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.employee-block {
  border-radius: 6px;
  padding: 6px 12px;
  margin-bottom: 2px;
  cursor: grab;
  user-select: none;
  transition: background 0.2s;
  font-weight: 500;
  text-align: center;
}
.employee-block.regular {
  background: #e0e7ef;
  border: 1px solid #7a8ba7;
}
.employee-block.parttime {
  background: #ffe6c7;
  border: 1px solid #e0a96d;
}
.employee-block:active {
  background: #b6c6e0;
}
.table-title {
  text-align: center;
  font-size: 1.5rem;
  margin-bottom: 16px;
  font-weight: bold;
}
.schedule-table-wrapper {
  flex: 1;
}
.schedule-table {
  border-collapse: collapse;
  width: 100%;
}
.schedule-table th,
.schedule-table td {
  border: 1px solid #333;
  text-align: center;
  min-width: 60px;
  height: 32px;
}
.drop-cell {
  min-width: 80px;
  height: 32px;
  background: #f8fafd;
  cursor: pointer;
  transition: background 0.2s;
}
.drop-cell:hover {
  background: #e6f0ff;
}
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 0, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}
.modal-content {
  background: #fff;
  padding: 32px 24px 24px 24px;
  border-radius: 12px;
  min-width: 320px;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.15);
}
.modal-content h3 {
  margin-top: 0;
  margin-bottom: 24px;
  font-size: 1.2rem;
  text-align: center;
}
.sortable-list {
  list-style: none;
  padding: 0;
  margin: 0 0 16px 0;
}
.modal-emp {
  padding: 6px 12px;
  border-radius: 6px;
  margin-bottom: 6px;
  background: #e0e7ef;
  border: 1px solid #7a8ba7;
  cursor: grab;
  display: flex;
  align-items: center;
  justify-content: space-between;
  transition: background 0.2s;
}
.modal-emp.parttime {
  background: #ffe6c7;
  border: 1px solid #e0a96d;
}
.modal-emp.selected {
  background: #b6c6e0;
}
.delete-btn {
  background: #e57373;
  color: #fff;
  border: none;
  border-radius: 4px;
  padding: 2px 10px;
  margin-left: 12px;
  cursor: pointer;
  font-size: 0.95em;
}
.delete-btn:hover {
  background: #b71c1c;
}
.modal-btns {
  display: flex;
  justify-content: center;
  gap: 16px;
}
.regular-bg {
  background: #e0e7ef !important;
  border: 1px solid #7a8ba7 !important;
}
.parttime-bg {
  background: #ffe6c7 !important;
  border: 1px solid #e0a96d !important;
}
.holiday-cell {
  color: #d32f2f !important;
  font-weight: bold;
}
.holiday-text {
  color: #d32f2f;
  font-weight: bold;
}
.checkmark {
  color: #1976d2;
  font-size: 1.2em;
  font-weight: bold;
}
.star-mark {
  color: #ffd700;
  font-size: 1.2em;
  font-weight: bold;
}
.employee-row {
  display: flex;
  gap: 6px;
  margin-bottom: 4px;
}
.input-section input[type='text'],
.input-section select {
  height: 40px;
  font-size: 1.1rem;
  padding: 0 12px;
  border-radius: 6px;
  border: 1px solid #b0b0b0;
  box-sizing: border-box;
}
.input-section input[type='text'] {
  min-width: 140px;
}
.input-section select {
  min-width: 110px;
}
.title-input {
  width: 150px;
  max-width: 180px;
  font-size: 1.1rem;
  text-align: center;
  border: none;
  border-bottom: 1px solid #888;
  outline: none;
  background: transparent;
  margin: 0 2px;
}
.date-display {
  font-size: 0.9em;
  font-weight: normal;
  color: #555;
}
.holiday-manage-modal {
  min-width: 420px;
  max-width: 520px;
  padding: 32px 20px 22px 20px;
}
.holiday-manage-table {
  width: 100%;
  border-collapse: separate;
  border-spacing: 0 2px;
  margin-bottom: 24px;
}
.holiday-manage-table th {
  background: #f5f7fa;
  font-size: 1.08rem;
  padding: 8px 0 8px 0;
  border-bottom: 2px solid #b0b0b0;
  text-align: center;
}
.holiday-manage-table td {
  padding: 8px 16px;
  min-width: 110px;
  text-align: center;
}
.hm-emp-name {
  font-weight: bold;
  color: #1976d2;
  padding-left: 0;
}
.hm-select {
  min-width: 90px;
  font-size: 1.13rem;
  padding: 7px 18px 7px 12px;
  border-radius: 7px;
  border: 1.5px solid #b0b0b0;
  background: #fafdff;
  margin-left: 0;
  margin-right: 0;
  box-shadow: 0 1px 4px rgba(25, 118, 210, 0.07);
  transition: border 0.2s;
}
.hm-select:focus {
  border: 1.5px solid #1976d2;
  outline: none;
}
.plan-list {
  list-style: none;
  padding: 0;
  margin: 0 0 16px 0;
  max-height: 400px;
  overflow-y: auto;
}
.plan-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  border-radius: 6px;
  margin-bottom: 6px;
  background: #f5f7fa;
  border: 1px solid #e0e7ef;
}
.plan-actions {
  display: flex;
  gap: 8px;
}
.plan-actions button {
  padding: 4px 12px;
  border-radius: 4px;
  border: 1px solid #b0b0b0;
  cursor: pointer;
  background: #fff;
}
.plan-actions .delete-btn {
  background: #fbe9e7;
  color: #d32f2f;
  border-color: #ffccbc;
}
/* (며칠전) 스타일 개선 */
.holiday-manage-table .reg-remain {
  margin-left: 8px;
  display: inline-block;
}
</style>
