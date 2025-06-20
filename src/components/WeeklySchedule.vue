<template>
  <div class="weekly-schedule">
    <!-- 화면에 보이는 UI -->
    <div class="screen-only">
      <!-- 표 타이틀 -->
      <h2 class="table-title">
        (
        <input
          v-model="titleBracketText"
          type="text"
          maxlength="10"
          placeholder="내용"
          class="title-input"
        />
        ) 주간근무계획
        <button class="confirm-btn" @click="confirmWeeklySchedule">
          주간근무계획 확정
        </button>
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
                <th>마지막 휴무 요일</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="emp in employees" :key="emp.name">
                <td class="hm-emp-name">{{ emp.name }}</td>
                <td>
                  <select
                    v-model="holidayManageMap[emp.name]"
                    @change="onHolidayManageChange(emp.name)"
                    class="hm-select"
                  >
                    <option v-for="(day, idx) in days" :key="day" :value="idx">
                      {{ day }}
                    </option>
                  </select>
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
                <th v-for="day in days" :key="day">{{ day }}</th>
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
                  @click="toggleCheckOrStar('A', idx, dayIdx)"
                  @dblclick="toggleHoliday('A', idx, dayIdx)"
                  :class="
                    emp && emp.holidays && emp.holidays[dayIdx]
                      ? 'holiday-cell'
                      : ''
                  "
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
                  @click="toggleCheckOrStar('B', idx, dayIdx)"
                  @dblclick="toggleHoliday('B', idx, dayIdx)"
                  :class="
                    emp && emp.holidays && emp.holidays[dayIdx]
                      ? 'holiday-cell'
                      : ''
                  "
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
      <h2 class="table-title">
        (
        {{ titleBracketText || ' ' }}
        ) 주간근무계획
      </h2>
      <table class="print-table">
        <thead>
          <tr>
            <th class="blank-header"></th>
            <th v-for="day in days" :key="'print-head-' + day">{{ day }}</th>
          </tr>
        </thead>
        <tbody>
          <tr>
            <th>A조</th>
            <td v-for="(day, dayIdx) in days" :key="'print-a-' + dayIdx">
              <div v-for="name in getWorkingEmployees('A', dayIdx)" :key="name">
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>B조</th>
            <td v-for="(day, dayIdx) in days" :key="'print-b-' + dayIdx">
              <div v-for="name in getWorkingEmployees('B', dayIdx)" :key="name">
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>마감</th>
            <td v-for="(day, dayIdx) in days" :key="'print-closer-' + dayIdx">
              <div v-for="name in getClosers(dayIdx)" :key="name">
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>휴무</th>
            <td v-for="(day, dayIdx) in days" :key="'print-holiday-' + dayIdx">
              <div v-for="name in getHolidayTakers(dayIdx)" :key="name">
                {{ name }}
              </div>
            </td>
          </tr>
          <tr>
            <th>내용</th>
            <td
              v-for="(day, dayIdx) in days"
              :key="'print-stats-' + dayIdx"
              class="stats-cell"
            >
              총원: {{ getDailyTotal(dayIdx) }}<br />
              근무: {{ getStats('total', '', dayIdx) }}<br />
              휴무: {{ getStats('holiday', '', dayIdx) }}<br />
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
  </div>
</template>

<script setup>
// 한글 주석: 필요한 Vue 기능 import
import { ref, computed, watch, onMounted } from 'vue';

// 한글 주석: 요일 배열
const days = ['일', '월', '화', '수', '목', '금', '토'];

// 한글 주석: 직원 입력 상태
const newEmployeeName = ref('');
const newEmployeeType = ref('정직원');

// 한글 주석: 전체 직원 목록 (이름, 구분)
const employees = ref([]);

// 한글 주석: 정직원/파트타임 분리 (근무표에 배치되지 않은 직원만)
const unassignedEmployees = computed(() => {
  // 근무표에 배치된 직원 이름 목록
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

// 한글 주석: A조/B조 직원(7명 고정, null로 채움, 휴일 정보 포함)
function makeTeamArray() {
  // 직원 객체에 holidays(요일별 휴일 여부) 추가
  return Array(7)
    .fill(null)
    .map(() => null);
}
const aTeam = ref(makeTeamArray());
const bTeam = ref(makeTeamArray());

// 한글 주석: 드래그 중인 직원 정보
const dragEmployee = ref(null);

// 한글 주석: 직원 추가 함수
function addEmployee() {
  // 에러 처리: 이름 미입력, 중복
  if (!newEmployeeName.value.trim()) {
    showMessageBoxWithText('이름을 입력하세요.');
    return;
  }
  if (
    employees.value.some((e) => e.name === newEmployeeName.value.trim()) ||
    aTeam.value.some((e) => e && e.name === newEmployeeName.value.trim()) ||
    bTeam.value.some((e) => e && e.name === newEmployeeName.value.trim())
  ) {
    showMessageBoxWithText('이미 등록된 이름입니다.');
    return;
  }
  employees.value.push({
    name: newEmployeeName.value.trim(),
    type: newEmployeeType.value,
  });
  newEmployeeName.value = '';
}

// 한글 주석: 드래그 시작
function onDragStart(emp) {
  dragEmployee.value = emp;
}

// 한글 주석: 직원명부 순서에 따라 조별로 자동 정렬하는 함수
function sortTeamByEmployeeOrder(team) {
  // 직원명부 내 순서와 정직원 > 파트타임 우선순위로 정렬, null은 뒤로
  const order = employees.value.map((e) => e.name);
  team.value = team.value
    .filter((e) => e)
    .sort((a, b) => {
      if (a.type !== b.type) return a.type === '정직원' ? -1 : 1;
      return order.indexOf(a.name) - order.indexOf(b.name);
    });
  while (team.value.length < 7) team.value.push(null);
}

// 한글 주석: 드롭 처리
function onDrop(teamType, idx) {
  if (!dragEmployee.value) return;
  // 근무표에 이미 배치된 직원은 무시
  if (
    aTeam.value.some((e) => e && e.name === dragEmployee.value.name) ||
    bTeam.value.some((e) => e && e.name === dragEmployee.value.name)
  )
    return;
  // 이름, 타입, 마지막 휴무 인덱스를 복사하고 근무기록은 모두 초기화
  const newEmp = {
    name: dragEmployee.value.name,
    type: dragEmployee.value.type,
    lastHolidayIndex: dragEmployee.value.lastHolidayIndex,
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

// 한글 주석: 관리 버튼(추후 구현)
function manageEmployees() {
  alert('관리 기능은 추후 구현 예정입니다.');
}

// 한글 주석: 셀에서 직원 더블클릭 시 좌측 목록으로 복귀
function onRemoveFromTeam(teamType, idx) {
  let emp = null;
  if (teamType === 'A') {
    emp = aTeam.value[idx];
    aTeam.value[idx] = null;
    sortTeamByEmployeeOrder(aTeam);
  } else {
    emp = bTeam.value[idx];
    bTeam.value[idx] = null;
    sortTeamByEmployeeOrder(bTeam);
  }
}

// 한글 주석: 모달 표시 상태
const showModal = ref(false);
const showPrintModal = ref(false);
// 한글 주석: 모달 내 직원 배열(정직 > 파트 순, 드래그/삭제/저장용)
const modalEmployees = ref([]);
// 한글 주석: 선택된 직원 인덱스
const selectedIdx = ref(null);

// 한글 주석: 모달 열 때 직원명부 복사 및 정렬
function openModal() {
  // employees(명부)만 복사 후 정직원 > 파트타임 순 정렬
  modalEmployees.value = employees.value
    .map((emp) => ({ ...emp }))
    .sort((a, b) => (a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1));
  selectedIdx.value = null;
  showModal.value = true;
}

function closeModal() {
  showModal.value = false;
}

// 한글 주석: 모달에서 드래그 시작
let dragIdx = null;
function onModalDragStart(idx) {
  dragIdx = idx;
}

// 한글 주석: 모달에서 드롭(순서 변경)
function onModalDrop(dropIdx) {
  if (dragIdx === null || dragIdx === dropIdx) return;
  const arr = modalEmployees.value;
  const moved = arr.splice(dragIdx, 1)[0];
  arr.splice(dropIdx, 0, moved);
  dragIdx = null;
  // 정직원 > 파트타임 순서로 정렬
  modalEmployees.value = [...modalEmployees.value].sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
}

// 한글 주석: 모달에서 직원 삭제
function deleteModalEmployee(idx) {
  modalEmployees.value.splice(idx, 1);
  selectedIdx.value = null;
  // 정직원 > 파트타임 순서로 정렬
  modalEmployees.value = [...modalEmployees.value].sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
}

// 한글 주석: 모달 저장(실제 직원 데이터 반영)
function saveModalEmployees() {
  // 저장 시 정직원 > 파트타임 순서로 자동 정렬
  const sorted = [...modalEmployees.value].sort((a, b) =>
    a.type === b.type ? 0 : a.type === '정직원' ? -1 : 1
  );
  employees.value = sorted;
  // 각 조 정렬만 새 순서에 맞게
  sortTeamByEmployeeOrder(aTeam);
  sortTeamByEmployeeOrder(bTeam);
  closeModal();
}

// 한글 주석: 출력 모달 열기
function openPrintModal() {
  showPrintModal.value = true;
}

// 한글 주석: 출력 모달 닫기
function closePrintModal() {
  showPrintModal.value = false;
}

// 한글 주석: A4 출력 기능
function printSchedule() {
  window.print();
  closePrintModal();
}

// 한글 주석: PDF 저장 기능 (브라우저 인쇄 기능 사용)
function downloadPDF() {
  window.print();
  closePrintModal();
}

// 한글 주석: 전체 직원명부(좌측+표에 배치된 직원 모두)
const employeesList = computed(() => {
  // 좌측 목록 + A조/B조에 배치된 직원까지 모두 합침
  const list = [...employees.value];
  aTeam.value.forEach((e) => {
    if (e) list.push(e);
  });
  bTeam.value.forEach((e) => {
    if (e) list.push(e);
  });
  return list;
});

// 한글 주석: 직원 데이터 localStorage 키
const STORAGE_KEY = 'daiso-schedule-employees-v1';

// 한글 주석: 타이틀 괄호 안 텍스트 상태
const titleBracketText = ref('');

// 한글 주석: 직원 데이터 저장 함수
function saveToStorage() {
  const data = {
    employees: employees.value,
    aTeam: aTeam.value,
    bTeam: bTeam.value,
    titleBracketText: titleBracketText.value,
  };
  localStorage.setItem(STORAGE_KEY, JSON.stringify(data));
}

// 한글 주석: 직원 데이터 불러오기 함수
function loadFromStorage() {
  const data = localStorage.getItem(STORAGE_KEY);
  if (data) {
    try {
      const parsed = JSON.parse(data);
      employees.value = parsed.employees || [];
      aTeam.value = Array.isArray(parsed.aTeam)
        ? parsed.aTeam
        : Array(7).fill(null);
      bTeam.value = Array.isArray(parsed.bTeam)
        ? parsed.bTeam
        : Array(7).fill(null);
      titleBracketText.value = parsed.titleBracketText || '';
    } catch (e) {
      // 에러 발생 시 초기화
      employees.value = [];
      aTeam.value = Array(7).fill(null);
      bTeam.value = Array(7).fill(null);
      titleBracketText.value = '';
    }
  }
}

// 한글 주석: employees, aTeam, bTeam, titleBracketText가 바뀔 때마다 저장
watch([employees, aTeam, bTeam, titleBracketText], saveToStorage, {
  deep: true,
});

// 한글 주석: 컴포넌트 마운트 시 데이터 불러오기
onMounted(loadFromStorage);

// 한글 주석: 메시지 박스 상태
const showMessageBox = ref(false);
const messageBoxText = ref('');
const messageBoxType = ref('default'); // 'default' | 'confirm'
let messageBoxCallback = null;

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

// 한글 주석: 주간근무계획 확정 함수
function confirmWeeklySchedule() {
  showMessageBoxWithText(
    '주간근무계획을 확정하시겠습니까? 휴무관리 기준이 갱신됩니다.',
    'confirm',
    () => {
      employees.value.forEach((masterEmp) => {
        // 근무표(A/B)에서 해당 직원 찾기
        const teamEmp =
          aTeam.value.find((e) => e && e.name === masterEmp.name) ||
          bTeam.value.find((e) => e && e.name === masterEmp.name);
        let lastIdx = -1;
        if (teamEmp && teamEmp.holidays) {
          for (let i = 0; i < teamEmp.holidays.length; i++) {
            if (teamEmp.holidays[i]) lastIdx = i;
          }
        }
        if (lastIdx !== -1) {
          masterEmp.lastHolidayIndex = lastIdx;
        } else {
          delete masterEmp.lastHolidayIndex;
        }
      });
      showMessageBoxWithText(
        '주간근무계획이 확정되었습니다. 휴무관리 기준이 갱신됩니다.'
      );
    }
  );
}

// 한글 주석: 휴일 토글 함수
function getDayNumber(weekType, dayIdx) {
  // weekType: 'this' | 'last', dayIdx: 0(일)~6(토)
  // 이번주: 토1~일7, 지난주: 토8~일14
  const base = weekType === 'this' ? 1 : 8;
  // dayIdx: 0(일)~6(토) → 토:6, 금:5, ... 일:0
  return base + ((6 - dayIdx + 7) % 7);
}

function toggleHoliday(teamType, idx, dayIdx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp) return;
  if (!emp.holidays) emp.holidays = Array(7).fill(false);
  const isHoliday = emp.holidays[dayIdx];
  const currentCount = emp.holidays.filter(Boolean).length;
  const maxHoliday = emp.type === '정직원' ? 2 : 1;

  // 휴무관리 요일 번호 (지난주)
  const lastWeekNum =
    typeof emp.lastHolidayIndex === 'number'
      ? getDayNumber('last', emp.lastHolidayIndex)
      : undefined;
  // 근무표 요일 번호 (이번주)
  const thisWeekNum = getDayNumber('this', dayIdx);

  if (emp.type === '정직원') {
    // 이번주에 이미 휴일이 있으면 그 인덱스가 기준(두번째 휴무)
    const firstHolidayIdx = emp.holidays.findIndex((h) => h);
    if (firstHolidayIdx !== -1 && !isHoliday) {
      // 두번째 휴무: 이번주 내에서만 비교
      const firstNum = getDayNumber('this', firstHolidayIdx);
      const diff = firstNum - thisWeekNum;
      if (diff > 5) {
        showMessageBoxWithText(
          '정직원은 기준 휴무일로부터 5일 이내만 휴무를 지정할 수 있습니다.'
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
      // 첫번째 휴무: 지난주 기준
      if (typeof lastWeekNum === 'number') {
        const diff = lastWeekNum - thisWeekNum;
        if (diff > 5) {
          showMessageBoxWithText(
            '정직원은 기준 휴무일로부터 5일 이내만 휴무를 지정할 수 있습니다.'
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
    }
  } else {
    // 파트타임
    if (!isHoliday) {
      if (currentCount >= 1) {
        showMessageBoxWithText(
          '파트타임은 한 주에 한 번만 휴일을 지정할 수 있습니다.'
        );
        return;
      }
      if (typeof lastWeekNum === 'number') {
        const diff = lastWeekNum - thisWeekNum;
        if (diff > 6) {
          showMessageBoxWithText(
            '파트타임은 기준 휴무일로부터 6일 이내만 휴무를 지정할 수 있습니다.'
          );
          return;
        }
      }
    }
  }
  emp.holidays[dayIdx] = !isHoliday;
  // 한글 주석: 근무표에서만 임시로 토글, 확정 전까지는 휴무관리 기준에 반영하지 않음
}

// 한글 주석: 체크와 별 토글 함수
function toggleCheckOrStar(teamType, idx, dayIdx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp) return;

  // 휴일인 경우 토글 불가
  if (emp.holidays && emp.holidays[dayIdx]) return;

  // 배열 초기화 (존재하지 않을 때만)
  if (!emp.checked) emp.checked = Array(7).fill(false);
  if (!emp.stars) emp.stars = Array(7).fill(false);

  // B조 정직원인 경우: 체크 ↔ 별 토글
  if (teamType === 'B' && emp.type === '정직원') {
    if (emp.checked[dayIdx]) {
      // 체크가 있으면 별로 변경
      emp.checked[dayIdx] = false;
      emp.stars[dayIdx] = true;
    } else if (emp.stars[dayIdx]) {
      // 별이 있으면 빈 셀로 변경 (체크로 바뀌지 않음)
      emp.stars[dayIdx] = false;
    } else {
      // 아무것도 없으면 체크로 설정
      emp.checked[dayIdx] = true;
    }
  } else {
    // A조 또는 파트타임: 체크만 토글
    emp.checked[dayIdx] = !emp.checked[dayIdx];
    emp.stars[dayIdx] = false; // 별표시 해제
  }
}

// 한글 주석: 성명란 클릭 시 해당 직원의 요일별 체크 토글
function onNameCellClick(teamType, idx) {
  const team = teamType === 'A' ? aTeam.value : bTeam.value;
  const emp = team[idx];
  if (!emp) return;

  // 배열 초기화
  if (!emp.checked) emp.checked = Array(7).fill(false);
  if (!emp.stars) emp.stars = Array(7).fill(false);
  if (!emp.holidays) emp.holidays = Array(7).fill(false);

  // 모든 직원: 빈 셀 ↔ 모든 셀 체크 토글 (휴일, 별표시는 유지)
  const checkedCells = emp.checked.filter(Boolean).length;

  if (checkedCells === 0) {
    // 체크가 없으면 모든 빈 셀에 체크 표시 (휴일, 별표시는 그대로)
    for (let i = 0; i < 7; i++) {
      if (!emp.holidays[i] && !emp.stars[i]) {
        emp.checked[i] = true;
      }
    }
  } else {
    // 체크가 있으면 모든 체크 해제 (휴일, 별표시는 그대로)
    emp.checked = Array(7).fill(false);
  }
}

// 한글 주석: 통계 함수
function getStats(teamType, empType, dayIdx) {
  let val = 0;
  if (teamType === 'A' || teamType === 'B') {
    let arr = teamType === 'A' ? aTeam.value : bTeam.value;
    if (empType === '정직원') {
      val = arr.reduce((acc, emp) => {
        if (
          emp &&
          emp.type === '정직원' &&
          !emp.holidays?.[dayIdx] &&
          (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
        ) {
          return acc + 1;
        }
        return acc;
      }, 0);
    } else if (empType === '파트타임') {
      val = arr.reduce((acc, emp) => {
        if (
          emp &&
          emp.type === '파트타임' &&
          !emp.holidays?.[dayIdx] &&
          emp.checked?.[dayIdx]
        ) {
          return acc + 1;
        }
        return acc;
      }, 0);
    } else if (empType === '계') {
      val = arr.reduce((acc, emp) => {
        if (
          emp &&
          !emp.holidays?.[dayIdx] &&
          (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
        ) {
          return acc + 1;
        }
        return acc;
      }, 0);
    }
  } else if (teamType === 'total') {
    let a = aTeam.value.reduce((acc, emp) => {
      if (
        emp &&
        !emp.holidays?.[dayIdx] &&
        (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
      ) {
        return acc + 1;
      }
      return acc;
    }, 0);
    let b = bTeam.value.reduce((acc, emp) => {
      if (
        emp &&
        !emp.holidays?.[dayIdx] &&
        (emp.checked?.[dayIdx] || emp.stars?.[dayIdx])
      ) {
        return acc + 1;
      }
      return acc;
    }, 0);
    val = a + b;
  } else if (teamType === 'holiday') {
    let a = aTeam.value.filter(
      (e) => e && e.holidays && e.holidays[dayIdx]
    ).length;
    let b = bTeam.value.filter(
      (e) => e && e.holidays && e.holidays[dayIdx]
    ).length;
    val = a + b;
  }
  return val === 0 ? '-' : val;
}

function chunkArray(arr, size) {
  const result = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}
const regularEmployeeRows = computed(() =>
  chunkArray(regularEmployees.value, 2)
);
const partTimeEmployeeRows = computed(() =>
  chunkArray(partTimeEmployees.value, 2)
);

// --- 출력용 데이터 계산 함수 ---
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
const getClosers = (dayIdx) => {
  return bTeam.value
    .filter((emp) => emp && emp.type === '정직원' && emp.stars?.[dayIdx])
    .map((emp) => emp.name);
};
const getHolidayTakers = (dayIdx) => {
  return [...aTeam.value, ...bTeam.value]
    .filter((emp) => emp && emp.holidays?.[dayIdx])
    .map((emp) => emp.name);
};
const getDailyTotal = (dayIdx) => {
  const working = getStats('total', '', dayIdx);
  const holiday = getStats('holiday', '', dayIdx);
  const total =
    (working === '-' ? 0 : working) + (holiday === '-' ? 0 : holiday);
  return total === 0 ? '-' : total;
};

// 한글 주석: 휴무관리 버튼 클릭 (아직 미구현)
function openHolidayManageModal() {
  // 직원별로 현재 lastHolidayIndex를 기본값으로 세팅
  holidayManageMap.value = {};
  employees.value.forEach((emp) => {
    holidayManageMap.value[emp.name] =
      typeof emp.lastHolidayIndex === 'number' ? emp.lastHolidayIndex : 6;
  });
  showHolidayManageModal.value = true;
}

// 한글 주석: 휴무관리 모달 상태
const showHolidayManageModal = ref(false);
const holidayManageMap = ref({});

function closeHolidayManageModal() {
  showHolidayManageModal.value = false;
}

function saveHolidayManageAll() {
  employees.value.forEach((emp) => {
    if (holidayManageMap.value.hasOwnProperty(emp.name)) {
      emp.lastHolidayIndex = Number(holidayManageMap.value[emp.name]);
    }
  });
  showMessageBoxWithText('모든 직원의 마지막 휴무 요일이 저장되었습니다.');
  closeHolidayManageModal();
}

function onHolidayManageChange(name) {
  const emp = employees.value.find((e) => e.name === name);
  if (emp) {
    emp.lastHolidayIndex = Number(holidayManageMap.value[name]);
  }
}
</script>

<style>
.print-only {
  display: none;
}
@media print {
  @page {
    size: A4 portrait;
    margin: 10mm;
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
  .print-table tbody tr:nth-child(1) th,
  .print-table tbody tr:nth-child(1) td {
    height: 45mm;
  }
  .print-table tbody tr:nth-child(2) th,
  .print-table tbody tr:nth-child(2) td {
    height: 45mm;
  }
  .print-table tbody tr:nth-child(3) th,
  .print-table tbody tr:nth-child(3) td {
    height: 25mm;
  }
  .print-table tbody tr:nth-child(4) th,
  .print-table tbody tr:nth-child(4) td {
    height: 30mm;
  }
  .print-table tbody tr:nth-child(5) th {
    height: 60mm;
    vertical-align: middle;
  }
  .print-table tbody tr:nth-child(5) td {
    height: 60mm;
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
  width: 80px;
  max-width: 120px;
  font-size: 1.1rem;
  text-align: center;
  border: none;
  border-bottom: 1px solid #888;
  outline: none;
  background: transparent;
  margin: 0 2px;
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
  padding: 4px 0 4px 0;
  font-size: 1.08rem;
  vertical-align: middle;
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
</style>
