<script setup lang="ts">
import { ref, computed } from 'vue'

interface Person {
  name: string
  type: 'student' | 'teacher'
  grade: number
}

const students = ref<Person[]>([])
const teachers = ref<Person[]>([])
const numberOfTables = ref(7) // Changed from ref(1) to ref(7)
const tables = ref<Person[][]>([])

const studentInput = ref('')
const teacherInput = ref('')
const activeTab = ref('rosters')

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
  } else {
    teachers.value = processedPeople
  }
}

const generateTables = () => {
  const grades = [6, 7, 8]
  const allTables: Record<number, Person[][]> = {}

  grades.forEach(grade => {
    const gradeStudents = students.value.filter(s => s.grade === grade)
    const gradeTeachers = teachers.value.filter(t => t.grade === grade)

    if (gradeStudents.length === 0 || gradeTeachers.length === 0) {
      allTables[grade] = []
      return
    }

    const gradeTables: Person[][] = Array.from({ length: numberOfTables.value }, () => [])

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
      gradeTables[index % numberOfTables.value].push(student)
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

  tables.value = allTables
}

const isGenerateDisabled = computed(() => {
  const hasAllGrades = [6, 7, 8].every(grade => 
    students.value.some(s => s.grade === grade) && 
    teachers.value.some(t => t.grade === grade)
  )
  return !hasAllGrades || numberOfTables.value < 1
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

  const sourceTableIndex = tables.value[grade].findIndex(table => 
    table.some(p => p.name === draggedPerson.value?.name && p.type === draggedPerson.value?.type)
  )

  if (sourceTableIndex === -1) return

  const sourceTable = tables.value[grade][sourceTableIndex]
  const targetTable = tables.value[grade][targetTableIndex]

  const personIndex = sourceTable.findIndex(p => p.name === draggedPerson.value?.name && p.type === draggedPerson.value?.type)
  const [movedPerson] = sourceTable.splice(personIndex, 1)
  targetTable.push(movedPerson)

  // Sort students alphabetically within the target table
  const teacher = targetTable.find(person => person.type === 'teacher')
  const sortedStudents = targetTable
    .filter(person => person.type === 'student')
    .sort((a, b) => a.name.localeCompare(b.name))
  tables.value[grade][targetTableIndex] = teacher ? [teacher, ...sortedStudents] : sortedStudents

  draggedPerson.value = null
}

const tableCards = computed(() => {
  const cards = []
  const grades = [6, 7, 8]
  
  for (let tableIndex = 0; tableIndex < numberOfTables.value; tableIndex++) {
    const card = {
      tableNumber: tableIndex + 1,
      gradeAssignments: {} as Record<number, Person[]>
    }
    
    grades.forEach(grade => {
      if (tables.value[grade] && tables.value[grade][tableIndex]) {
        card.gradeAssignments[grade] = tables.value[grade][tableIndex]
      }
    })
    
    cards.push(card)
  }
  
  return cards
})

const studentLists = computed(() => {
  const lists: Record<number, { name: string; tableNumber: number }[]> = {
    6: [],
    7: [],
    8: []
  }

  Object.entries(tables.value).forEach(([grade, gradeTables]) => {
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
</script>

<template>
  <div>
    <h1>Table Assignment Generator</h1>
    
    <div class="tabs">
      <button @click="activeTab = 'rosters'" :class="{ active: activeTab === 'rosters' }">Rosters</button>
      <button @click="activeTab = 'assignments'" :class="{ active: activeTab === 'assignments' }">Table Assignments</button>
      <button @click="activeTab = 'cards'" :class="{ active: activeTab === 'cards' }">Table Cards</button>
      <button @click="activeTab = 'studentLists'" :class="{ active: activeTab === 'studentLists' }">Student Lists</button>
    </div>

    <div v-if="activeTab === 'rosters'">
      <div>
        <label>
          Students (name and grade, tab-separated, one per line):
          <textarea v-model="studentInput" rows="5" cols="30" placeholder="John Doe&#9;9"></textarea>
        </label>
        <button @click="importPeople('student')">Import Students</button>
      </div>
      <div>
        <label>
          Teachers (name and grade, tab-separated, one per line):
          <textarea v-model="teacherInput" rows="5" cols="30" placeholder="Jane Smith&#9;10"></textarea>
        </label>
        <button @click="importPeople('teacher')">Import Teachers</button>
      </div>
    </div>

    <div v-if="activeTab === 'assignments'">
      <div>
        <label>
          Number of Tables:
          <input type="number" v-model="numberOfTables" min="1">
        </label>
      </div>
      <button @click="generateTables" :disabled="isGenerateDisabled">Generate Tables</button>
      
      <div v-if="Object.keys(tables).length">
        <h2>Generated Tables</h2>
        <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-section">
          <h3>Grade {{ grade }}</h3>
          <div class="tables-grid">
            <div 
              v-for="(table, index) in tables[grade]" 
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
      <h2>Table Cards</h2>
      <div class="table-cards-container">
        <div v-for="card in tableCards" :key="card.tableNumber" class="table-card-print">
          <h3 class="table-number">Table {{ card.tableNumber }}</h3>
          <div class="grades-row">
            <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-list">
              <h4>Grade {{ grade }}</h4>
              <ul>
                <li v-for="person in card.gradeAssignments[grade]" :key="person.name">
                  <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                  <span v-else>{{ person.name }}</span>
                </li>
              </ul>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div v-if="activeTab === 'studentLists'">
      <h2>Student Lists</h2>
      <div class="student-lists-container">
        <div v-for="grade in [6, 7, 8]" :key="grade" class="grade-list-print">
          <h3>Grade {{ grade }}</h3>
          <div class="student-list">
            <div v-for="student in studentLists[grade]" :key="student.name" class="student-item">
              {{ student.name }} - Table {{ student.tableNumber }}
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
  height: 8.5in;
  border: 1px solid #000;
  margin: 0.25in;
  padding: 0.5in;
  box-sizing: border-box;
  page-break-inside: avoid;
  display: flex;
  flex-direction: column;
}

.table-number {
  font-size: 24px;
  text-align: center;
  margin-bottom: 1rem;
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
}

.grade-list:first-child {
  border-left: none;
  padding-left: 0;
}

.grade-list h4 {
  margin-top: 0;
  margin-bottom: 0.5rem;
  text-align: center;
}

.grade-list ul {
  margin: 0;
  padding-left: 1rem;
}

@media print {
  body * {
    visibility: hidden;
  }
  .table-cards-container, .table-cards-container * {
    visibility: visible;
  }
  .table-cards-container {
    position: absolute;
    left: 0;
    top: 0;
  }
}

.student-lists-container {
  column-width: 300px;
  column-gap: 2rem;
}

.grade-list-print {
  break-inside: avoid;
  page-break-inside: avoid;
  margin-bottom: 2rem;
}

.student-list {
  display: flex;
  flex-direction: column;
}

.student-item {
  margin-bottom: 0.5rem;
}

@media print {
  body * {
    visibility: hidden;
  }
  .student-lists-container, .student-lists-container * {
    visibility: visible;
  }
  .student-lists-container {
    position: absolute;
    left: 0;
    top: 0;
  }
}
</style>
