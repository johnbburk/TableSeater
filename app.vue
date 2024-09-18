<script setup lang="ts">
import { ref, computed, onMounted, watch } from 'vue'

interface Person {
  name: string
  type: 'student' | 'teacher'
  grade: number
}

const students = ref<Person[]>([])
const teachers = ref<Person[]>([])
const numberOfRotations = ref(6)
const currentRotation = ref(1)
const tables = ref<Record<number, Record<number, Person[][]>>>({})

const studentInput = ref('')
const teacherInput = ref('')
const activeTab = ref('rosters')
const printMode = ref(false)

// New: Number of tables for each grade
const tablesPerGrade = ref({
  6: 8,
  7: 9,
  8: 10
})

// Load data from local storage on component mount
onMounted(() => {
  const savedStudents = localStorage.getItem('students')
  const savedTeachers = localStorage.getItem('teachers')
  if (savedStudents) {
    students.value = JSON.parse(savedStudents)
    studentInput.value = students.value.map(s => `${s.name}\t${s.grade}`).join('\n')
  }
  if (savedTeachers) {
    teachers.value = JSON.parse(savedTeachers)
    teacherInput.value = teachers.value.map(t => `${t.name}\t${t.grade}`).join('\n')
  }
})

// Save data to local storage when it changes
watch(students, (newStudents) => {
  localStorage.setItem('students', JSON.stringify(newStudents))
}, { deep: true })

watch(teachers, (newTeachers) => {
  localStorage.setItem('teachers', JSON.stringify(newTeachers))
}, { deep: true })

const importPeople = (type: 'student' | 'teacher') => {
  const input = (type === 'student' ? studentInput.value : teacherInput.value)
    .split('\n')
    .filter(line => line.trim() !== '')
  
  const processedPeople = input.map(line => {
    const [name, grade] = line.split('\t');
    return {
      name: name.trim(),
      type,
      grade: parseInt(grade.trim(), 10)
    };
  });

  if (type === 'student') {
    students.value = processedPeople
    studentInput.value = students.value.map(s => `${s.name}\t${s.grade}`).join('\n')
  } else {
    teachers.value = processedPeople
    teacherInput.value = teachers.value.map(t => `${t.name}\t${t.grade}`).join('\n')
  }
}

const generateTables = () => {
  const grades = [6, 7, 8]
  const allRotations: Record<number, Record<number, Person[][]>> = {}

  for (let rotation = 1; rotation <= numberOfRotations.value; rotation++) {
    const allTables: Record<number, Person[][]> = {}

    grades.forEach(grade => {
      const gradeStudents = students.value.filter(s => s.grade === grade)
      const gradeTeachers = teachers.value.filter(t => t.grade === grade)

      if (gradeStudents.length === 0) {
        allTables[grade] = []
        return
      }

      const numberOfTables = tablesPerGrade.value[grade as keyof typeof tablesPerGrade.value]
      const gradeTables: Person[][] = Array.from({ length: numberOfTables }, () => [])

      // Randomly assign teachers
      const shuffledTeachers = [...gradeTeachers].sort(() => Math.random() - 0.5)
      gradeTables.forEach((table, index) => {
        if (index < shuffledTeachers.length) {
          table.push(shuffledTeachers[index])
        }
      })

      // Randomly assign students
      const shuffledStudents = [...gradeStudents].sort(() => Math.random() - 0.5)
      shuffledStudents.forEach((student, index) => {
        gradeTables[index % numberOfTables].push(student)
      })

      // Sort students alphabetically within each table
      allTables[grade] = gradeTables.map(table => {
        const teacher = table.find(person => person.type === 'teacher')
        const sortedStudents = table
          .filter(person => person.type === 'student')
          .sort((a, b) => a.name.localeCompare(b.name))
        return teacher ? [teacher, ...sortedStudents] : sortedStudents
      })
    })

    allRotations[rotation] = allTables
  }

  tables.value = allRotations
  currentRotation.value = 1 // Reset to first rotation after generating
}

const isGenerateDisabled = computed(() => {
  return !students.value.length || Object.values(tablesPerGrade.value).some(num => num < 1)
})

const draggedPerson = ref<Person | null>(null)

const onDragStart = (person: Person, grade: number, tableIndex: number) => {
  draggedPerson.value = person
}

const onDragOver = (event: DragEvent) => {
  event.preventDefault()
}

const onDrop = (grade: number, targetTableIndex: number) => {
  if (!draggedPerson.value) return

  const sourceTableIndex = tables.value[currentRotation.value][grade].findIndex(table => 
    table.some(s => s.name === draggedPerson.value?.name)
  )

  if (sourceTableIndex === -1) return

  const sourceTable = tables.value[currentRotation.value][grade][sourceTableIndex]
  const targetTable = tables.value[currentRotation.value][grade][targetTableIndex]

  const personIndex = sourceTable.findIndex(s => s.name === draggedPerson.value?.name)
  const [movedPerson] = sourceTable.splice(personIndex, 1)
  targetTable.push(movedPerson)

  // Sort students alphabetically within the target table
  tables.value[currentRotation.value][grade][targetTableIndex] = targetTable.sort((a, b) => a.name.localeCompare(b.name))

  draggedPerson.value = null
}

const tableCards = computed(() => {
  const cards = []
  const grades = [6, 7, 8]
  const maxTables = Math.max(...Object.values(tablesPerGrade.value))
  
  for (let tableNumber = 1; tableNumber <= maxTables; tableNumber++) {
    const card = {
      tableNumber,
      grades: {} as Record<number, Person[]>
    }

    for (let grade of grades) {
      if (tableNumber <= tablesPerGrade.value[grade as keyof typeof tablesPerGrade.value]) {
        const people = tables.value[currentRotation.value]?.[grade]?.[tableNumber - 1] || []
        if (people.length > 0) {
          card.grades[grade] = people
        }
      }
    }

    if (Object.keys(card.grades).length > 0) {
      cards.push(card)
    }
  }
  
  return cards
})

const studentLists = computed(() => {
  const lists: Record<number, { name: string; tableNumber: number }[]> = {
    6: [],
    7: [],
    8: []
  }

  if (!tables.value[currentRotation.value]) return lists

  Object.entries(tables.value[currentRotation.value]).forEach(([grade, gradeTables]) => {
    gradeTables.forEach((table, tableIndex) => {
      table.forEach(person => {
        if (person.type === 'student') {
          lists[parseInt(grade)].push({
            name: person.name,
            tableNumber: tableIndex + 1
          })
        }
      })
    })
  })

  // Sort each grade's list alphabetically
  Object.values(lists).forEach(gradeList => {
    gradeList.sort((a, b) => a.name.localeCompare(b.name))
  })

  return lists
})

const resetList = (type: 'student' | 'teacher') => {
  if (type === 'student') {
    students.value = []
    studentInput.value = ''
    localStorage.removeItem('students')
  } else {
    teachers.value = []
    teacherInput.value = ''
    localStorage.removeItem('teachers')
  }
}

const printTableCards = () => {
  printMode.value = true
  setTimeout(() => {
    window.print()
    printMode.value = false
  }, 100)
}

const printStudentLists = () => {
  printMode.value = true
  setTimeout(() => {
    window.print()
    printMode.value = false
  }, 100)
}
</script>

<template>
  <div :class="{ 'print-content': printMode }">
    <div class="no-print">
      <h1>Table Assignment Generator</h1>
      
      <div class="rotation-selector" v-if="Object.keys(tables).length">
        <label for="rotation-select">Select Rotation:</label>
        <select id="rotation-select" v-model="currentRotation">
          <option v-for="rotation in numberOfRotations" :key="rotation" :value="rotation">
            Rotation {{ rotation }}
          </option>
        </select>
      </div>

      <div class="tabs">
        <button @click="activeTab = 'rosters'" :class="{ active: activeTab === 'rosters' }">Rosters</button>
        <button @click="activeTab = 'assignments'" :class="{ active: activeTab === 'assignments' }">Table Assignments</button>
        <button @click="activeTab = 'cards'" :class="{ active: activeTab === 'cards' }">Table Cards</button>
        <button @click="activeTab = 'studentLists'" :class="{ active: activeTab === 'studentLists' }">Student Lists</button>
      </div>
    </div>

    <div v-if="activeTab === 'rosters'">
      <div>
        <label>
          Students (name and grade, tab-separated, one per line):
          <textarea v-model="studentInput" rows="5" cols="30" placeholder="John Doe&#9;9"></textarea>
        </label>
        <div>
          <button @click="importPeople('student')">Import Students</button>
          <button @click="resetList('student')" class="reset-button">Reset Students</button>
        </div>
      </div>
      <div>
        <label>
          Teachers (name and grade, tab-separated, one per line):
          <textarea v-model="teacherInput" rows="5" cols="30" placeholder="Jane Smith&#9;10"></textarea>
        </label>
        <div>
          <button @click="importPeople('teacher')">Import Teachers</button>
          <button @click="resetList('teacher')" class="reset-button">Reset Teachers</button>
        </div>
      </div>
    </div>

    <div v-if="activeTab === 'assignments'">
      <div>
        <label v-for="grade in [6, 7, 8]" :key="grade">
          Number of Tables for Grade {{ grade }}:
          <input type="number" v-model="tablesPerGrade[grade]" min="1">
        </label>
      </div>
      <div>
        <label>
          Number of Rotations:
          <input type="number" v-model="numberOfRotations" min="1">
        </label>
      </div>
      <button @click="generateTables" :disabled="isGenerateDisabled">Generate Tables</button>
      
      <div v-if="Object.keys(tables).length">
        <h2>Generated Tables - Rotation {{ currentRotation }}</h2>
        <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-section">
          <h3>Grade {{ grade }}</h3>
          <div class="tables-grid">
            <div 
              v-for="(table, index) in tables[currentRotation][grade]" 
              :key="`${grade}-${index}`" 
              class="table-card"
              @dragover="onDragOver"
              @drop="onDrop(grade, index)"
            >
              <h4>Table {{ index + 1 }}</h4>
              <ul>
                <li 
                  v-for="person in table" 
                  :key="person.name"
                  draggable="true"
                  @dragstart="onDragStart(person, grade, index)"
                >
                  <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                  <span v-else>{{ person.name }}</span>
                </li>
              </ul>
            </div>
          </div>
          <hr v-if="grade !== 8" class="grade-separator">
        </div>
      </div>
    </div>

    <div v-if="activeTab === 'cards'">
      <h2 class="no-print">Table Cards - Rotation {{ currentRotation }}</h2>
      <button @click="printTableCards" class="print-button no-print">Print Table Cards</button>
      <div class="table-cards-container">
        <div v-for="card in tableCards" :key="card.tableNumber" class="table-card-print">
          <h3 class="table-number">Table {{ card.tableNumber }}</h3>
          <div class="grades-row">
            <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-list">
              <template v-if="card.grades[grade]">
                <h5>Grade {{ grade }}</h5>
                <ul>
                  <li v-for="person in card.grades[grade]" :key="person.name">
                    <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                    <span v-else>{{ person.name }}</span>
                  </li>
                </ul>
              </template>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="activeTab === 'studentLists'">
      <h2 class="no-print">Student Lists</h2>
      <button @click="printStudentLists" class="print-button no-print">Print Student Lists</button>
      <div class="student-lists-container">
        <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-list-print">
          <h3>Grade {{ grade }}</h3>
          <div class="student-list">
            <div v-for="student in studentLists[grade]" :key="student.name" class="student-item">
              {{ student.name }} - ({{ student.tableNumber }})
            </div>
          </div>
          <hr v-if="grade !== 8" class="grade-separator">
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
div {
  margin-bottom: 1rem;
}
label {
  display: block;
  margin-bottom: 0.5rem;
}
button {
  margin-top: 1rem;
}
ul {
  list-style-type: none;
  padding-left: 0;
}
textarea {
  width: 100%;
  margin-bottom: 0.5rem;
}
.tabs {
  display: flex;
  margin-bottom: 1rem;
}
.tabs button {
  padding: 0.5rem 1rem;
  margin-right: 0.5rem;
  border: 1px solid #ccc;
  background-color: #f0f0f0;
  cursor: pointer;
}
.tabs button.active {
  background-color: #fff;
  border-bottom: none;
}

.grade-section {
  margin-bottom: 2rem;
}

.tables-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}

.table-card {
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 1rem;
  min-height: 100px; /* Add this to ensure empty tables have a drop area */
}

li {
  cursor: move;
  padding: 0.25rem;
  border-radius: 4px;
}

li:hover {
  background-color: #f0f0f0;
}

.grade-separator {
  margin: 2rem 0;
  border: none;
  border-top: 1px solid #ccc;
}

.grade-tables {
  display: flex;
  justify-content: space-between;
}

.grade-column {
  flex: 1;
  margin: 0 10px;
}

.table-cards-container {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-around;
}

.table-card-print {
  width: 5.5in;
  height: 4.25in; /* Changed to half of 8.5in */
  border: 1px solid #000;
  margin: 0.125in;
  padding: 0.25in;
  box-sizing: border-box;
  page-break-inside: avoid;
  display: flex;
  flex-direction: column;
  position: relative; /* Add this */
  overflow: hidden; /* Add this */
}

.table-card-print::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: url('/logo.png');
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  opacity: 0.1;
  z-index: -1;
}

.table-number {
  font-size: 20px; /* Reduced font size */
  text-align: center;
  margin-bottom: 0.5rem;
}

.grades-row {
  display: flex;
  justify-content: space-between;
  flex-grow: 1;
}

.grade-list {
  flex: 1;
  margin: 0 0.25rem;
  border-left: 1px solid #ccc;
  padding-left: 0.5rem;
  min-width: 33%; /* Ensure each grade takes up at least a third of the space */
}

.grade-list:first-child {
  border-left: none;
  padding-left: 0;
}

.grade-list h5 {
  font-size: 16px; /* Reduced font size */
  margin-top: 0;
  margin-bottom: 0.25rem;
  text-align: center;
}

.grade-list ul {
  margin: 0;
  padding-left: 0.5rem;
  font-size: 12px; /* Reduced font size */
}

.grade-list li {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

@media print {
  .no-print {
    display: none !important;
  }
  
  .print-content {
    visibility: visible;
    position: absolute;
    left: 0;
    top: 0;
    width: 100%;
  }

  .table-cards-container, .student-lists-container {
    visibility: visible;
  }

  .table-cards-container {
    display: flex;
    flex-wrap: wrap;
  }

  .table-card-print {
    page-break-inside: avoid;
    break-inside: avoid;
  }

  .student-lists-container {
    display: flex;
    font-size: 8pt; /* Even smaller font for print */
    line-height: 1.1; /* Tighter line height for print */
  }

  .grade-list-print {
    flex: 1;
    break-inside: avoid;
    page-break-inside: avoid;
    margin-bottom: 1rem; /* Reduced margin */
  }

  .grade-list-print:nth-child(3) {
    flex: 1;
  }

  .grade-list-print h3 {
    font-size: 10pt; /* Smaller heading for print */
  }

  .student-list {
    column-count: 1;
    column-gap: 0.5rem;
  }

  .grade-list-print:nth-child(3) .student-list {
    column-count: 1;
  }

  .student-item {
    margin-bottom: 0.1rem; /* Further reduced margin for print */
  }

  /* Ensure the container takes up the full page */
  .print-content {
    width: 100%;
    height: 100vh;
    padding: 0.5in; /* Add some padding around the edges */
    box-sizing: border-box;
  }

  .table-card-print::before {
    -webkit-print-color-adjust: exact;
    print-color-adjust: exact;
  }
}

.student-lists-container {
  display: flex;
  font-size: 10px; /* Reduced font size */
  line-height: 1.2; /* Tightened line height */
}

.grade-list-print {
  break-inside: avoid;
  page-break-inside: avoid;
  margin-bottom: 1rem; /* Reduced margin */
  display: inline-block;
  width: 100%;
}

.grade-list-print h3 {
  font-size: 14px; /* Reduced heading size */
  margin-bottom: 0.5rem; /* Reduced margin */
}

.student-list {
  column-count: 1;
  column-gap: 1rem;
}

.grade-list-print:nth-child(3) .student-list {
  column-count: 1;
}

.student-item {
  break-inside: avoid;
  page-break-inside: avoid;
}

.reset-button {
  margin-left: 1rem;
  background-color: #f44336;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.reset-button:hover {
  background-color: #d32f2f;
}

.rotation-selector {
  margin-bottom: 1rem;
}

.rotation-selector label {
  margin-right: 0.5rem;
}

.rotation-selector select {
  padding: 0.25rem;
}

.print-button {
  margin-bottom: 1rem;
  background-color: #4CAF50;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  cursor: pointer;
}

.print-button:hover {
  background-color: #45a049;
}
</style>
