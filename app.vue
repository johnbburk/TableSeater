<template>
  <v-app>
    <v-main>
      <v-container fluid class="pa-4">
        <div :class="{ 'print-content': printMode }">
          <div class="no-print">
            <h1>Table Assignment Generator</h1>
            
            <div class="rotation-selector" v-if="Object.keys(tables).length">
              <v-select
                v-model="currentRotation"
                :items="rotationItems"
                label="Select Rotation"
                dense
              ></v-select>
            </div>

            <div class="tabs">
              <v-btn @click="activeTab = 'rosters'" :class="{ active: activeTab === 'rosters' }">Rosters</v-btn>
              <v-btn @click="activeTab = 'assignments'" :class="{ active: activeTab === 'assignments' }">Table Assignments</v-btn>
              <v-btn @click="activeTab = 'cards'" :class="{ active: activeTab === 'cards' }">Table Cards</v-btn>
              <v-btn @click="activeTab = 'studentLists'" :class="{ active: activeTab === 'studentLists' }">Student Lists</v-btn>
            </div>
          </div>

          <div v-if="activeTab === 'rosters'">
            <v-card class="mb-4">
              <v-card-text>
                <v-textarea
                  v-model="studentInput"
                  label="Students (name and grade, tab-separated, one per line)"
                  rows="5"
                  placeholder="John Doe&#9;9"
                ></v-textarea>
              </v-card-text>
              <v-card-actions>
                <v-btn @click="importPeople('student')" color="primary">Import Students</v-btn>
                <v-btn @click="resetList('student')" color="error">Reset Students</v-btn>
              </v-card-actions>
            </v-card>

            <v-card>
              <v-card-text>
                <v-textarea
                  v-model="teacherInput"
                  label="Teachers (name and grade, tab-separated, one per line)"
                  rows="5"
                  placeholder="Jane Smith&#9;10"
                ></v-textarea>
              </v-card-text>
              <v-card-actions>
                <v-btn @click="importPeople('teacher')" color="primary">Import Teachers</v-btn>
                <v-btn @click="resetList('teacher')" color="error">Reset Teachers</v-btn>
              </v-card-actions>
            </v-card>
          </div>

          <div v-if="activeTab === 'assignments'">
            <v-switch
              v-model="seatingMode"
              :label="`${seatingMode === 'whole' ? 'Whole School Seating' : 'Individual Grades Seating'}`"
              true-value="whole"
              false-value="individual"
              class="mb-4"
            ></v-switch>

            <div v-if="seatingMode === 'whole'">
              <v-text-field
                v-model="totalTables"
                type="number"
                label="Total Number of Tables"
                min="1"
              ></v-text-field>
            </div>

            <div v-else>
              <v-text-field
                v-for="grade in [6, 7, 8]"
                :key="grade"
                v-model="tablesPerGrade[grade]"
                type="number"
                :label="`Number of Tables for Grade ${grade}`"
                min="1"
              ></v-text-field>
            </div>

            <div v-if="seatingMode === 'individual'">
              <v-text-field
                v-model="numberOfRotations"
                type="number"
                label="Number of Rotations"
                min="1"
              ></v-text-field>
            </div>

            <v-btn @click="generateTables" :disabled="isGenerateDisabled" color="primary">Generate Tables</v-btn>
            
            <div v-if="Object.keys(tables).length">
              <h2>Generated Tables{{ seatingMode === 'individual' ? ` - Rotation ${currentRotation}` : '' }}</h2>
              <div v-if="seatingMode === 'whole'" class="tables-grid">
                <div 
                  v-for="(table, index) in tables[1][0]" 
                  :key="index" 
                  class="table-card"
                  @dragover="onDragOver"
                  @drop="onDrop(0, index)"
                >
                  <h4>Table {{ index + 1 }}</h4>
                  <ul>
                    <li 
                      v-for="person in table" 
                      :key="person.name"
                      draggable="true"
                      @dragstart="onDragStart(person, 0, index)"
                    >
                      <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                      <span v-else>{{ person.name }} (Grade {{ person.grade }})</span>
                    </li>
                  </ul>
                </div>
              </div>
              <div v-else>
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
          </div>

          <div v-if="activeTab === 'cards'">
            <h2 class="no-print">Table Cards - Rotation {{ currentRotation }}</h2>
            
            <v-text-field
              v-model="tableCardNote"
              label="Table Card Note"
              class="no-print"
            ></v-text-field>
            
            <v-btn @click="printTableCards" color="primary" class="print-button no-print">Print Table Cards</v-btn>
            <div class="table-cards-container">
              <div v-for="card in tableCards" :key="card.tableNumber" class="table-card-print">
                <h3 class="table-number">Table {{ card.tableNumber }}</h3>
                <p v-if="tableCardNote" class="table-card-note">{{ tableCardNote }}</p>
                <ul class="people-list">
                  <li v-for="(person, index) in card.people" :key="index">
                    <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                    <span v-else>{{ person.name }} (Grade {{ person.grade }})</span>
                  </li>
                </ul>
              </div>
            </div>
          </div>

          <div v-if="activeTab === 'studentLists'">
            <h2 class="no-print">Student Lists</h2>
            <v-btn @click="printStudentLists" color="primary" class="print-button no-print">Print Student Lists</v-btn>
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
      </v-container>
    </v-main>
  </v-app>
</template>

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

const tableCardNote = ref('')

// Add these new refs
const seatingMode = ref('individual') // 'individual' or 'whole'
const totalTables = ref(27) // Default value, adjust as needed

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
  const savedNote = localStorage.getItem('tableCardNote')
  if (savedNote) {
    tableCardNote.value = savedNote
  }
})

// Save data to local storage when it changes
watch(students, (newStudents) => {
  localStorage.setItem('students', JSON.stringify(newStudents))
}, { deep: true })

watch(teachers, (newTeachers) => {
  localStorage.setItem('teachers', JSON.stringify(newTeachers))
}, { deep: true })

// Add this to save the note to local storage
watch(tableCardNote, (newNote) => {
  localStorage.setItem('tableCardNote', newNote)
})

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

  const rotationsToGenerate = seatingMode.value === 'whole' ? 1 : numberOfRotations.value

  for (let rotation = 1; rotation <= rotationsToGenerate; rotation++) {
    const allTables: Record<number, Person[][]> = {}

    if (seatingMode.value === 'whole') {
      // Whole School Seating logic
      const allTeachers = teachers.value.sort(() => Math.random() - 0.5)
      const allStudents = students.value.sort((a, b) => b.grade - a.grade || a.name.localeCompare(b.name))
      
      allTables[0] = Array.from({ length: totalTables.value }, () => [])

      // Distribute teachers
      allTeachers.forEach((teacher, index) => {
        allTables[0][index % totalTables.value].push(teacher)
      })

      // Distribute students
      allStudents.forEach((student, index) => {
        allTables[0][index % totalTables.value].push(student)
      })

      // Sort people within each table: teachers first, then students by grade (descending) and name
      allTables[0] = allTables[0].map(table => {
        const teachers = table.filter(person => person.type === 'teacher')
        const students = table.filter(person => person.type === 'student')
          .sort((a, b) => b.grade - a.grade || a.name.localeCompare(b.name))
        return [...teachers, ...students]
      })
    } else {
      // Individual Grades Seating logic
      grades.forEach(grade => {
        const gradeStudents = students.value.filter(s => s.grade === grade)
        const gradeTeachers = teachers.value.filter(t => t.grade === grade)

        if (gradeStudents.length === 0) {
          allTables[grade] = []
          return
        }

        const numberOfTables = tablesPerGrade.value[grade as keyof typeof tablesPerGrade.value]
        const gradeTables: Person[][] = Array.from({ length: numberOfTables }, () => [])

        // Distribute teachers
        gradeTeachers.forEach((teacher, index) => {
          gradeTables[index % numberOfTables].push(teacher)
        })

        // Distribute students
        gradeStudents
          .sort((a, b) => a.name.localeCompare(b.name))
          .forEach((student, index) => {
            gradeTables[index % numberOfTables].push(student)
          })

        // Sort people within each table: teachers first, then students by name
        allTables[grade] = gradeTables.map(table => {
          const teachers = table.filter(person => person.type === 'teacher')
          const students = table.filter(person => person.type === 'student')
            .sort((a, b) => a.name.localeCompare(b.name))
          return [...teachers, ...students]
        })
      })
    }

    allRotations[rotation] = allTables
  }

  tables.value = allRotations
  currentRotation.value = 1 // Reset to first rotation after generating
}

const isGenerateDisabled = computed(() => {
  if (seatingMode.value === 'whole') {
    return !students.value.length || totalTables.value < 1
  } else {
    return !students.value.length || Object.values(tablesPerGrade.value).some(num => num < 1)
  }
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

  const rotation = seatingMode.value === 'whole' ? 1 : currentRotation.value
  const sourceTableIndex = tables.value[rotation][grade].findIndex(table => 
    table.some(s => s.name === draggedPerson.value?.name)
  )

  if (sourceTableIndex === -1) return

  const sourceTable = tables.value[rotation][grade][sourceTableIndex]
  const targetTable = tables.value[rotation][grade][targetTableIndex]

  const personIndex = sourceTable.findIndex(s => s.name === draggedPerson.value?.name)
  const [movedPerson] = sourceTable.splice(personIndex, 1)
  targetTable.push(movedPerson)

  // Sort people within the target table
  if (seatingMode.value === 'whole') {
    tables.value[rotation][grade][targetTableIndex] = targetTable.sort((a, b) => {
      if (a.type !== b.type) return a.type === 'teacher' ? -1 : 1
      if (a.grade !== b.grade) return b.grade - a.grade
      return a.name.localeCompare(b.name)
    })
  } else {
    tables.value[rotation][grade][targetTableIndex] = targetTable.sort((a, b) => a.name.localeCompare(b.name))
  }

  draggedPerson.value = null
}

const tableCards = computed(() => {
  const cards = []
  const grades = seatingMode.value === 'whole' ? [0] : [6, 7, 8]
  const maxTables = seatingMode.value === 'whole' ? totalTables.value : Math.max(...Object.values(tablesPerGrade.value))
  
  for (let tableNumber = 1; tableNumber <= maxTables; tableNumber++) {
    const card = {
      tableNumber,
      people: [] as Person[]
    }

    if (seatingMode.value === 'whole') {
      card.people = tables.value[currentRotation.value]?.[0]?.[tableNumber - 1] || []
    } else {
      for (let grade of grades) {
        if (tableNumber <= tablesPerGrade.value[grade as keyof typeof tablesPerGrade.value]) {
          const tablePeople = tables.value[currentRotation.value]?.[grade]?.[tableNumber - 1] || []
          card.people.push(...tablePeople)
        }
      }
    }

    if (card.people.length > 0) {
      // Sort people: teachers first, then students by grade (descending) and name
      card.people.sort((a, b) => {
        if (a.type !== b.type) return a.type === 'teacher' ? -1 : 1
        if (a.grade !== b.grade) return b.grade - a.grade
        return a.name.localeCompare(b.name)
      })
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

  if (seatingMode.value === 'whole') {
    tables.value[currentRotation.value][0].forEach((table, tableIndex) => {
      table.forEach(person => {
        if (person.type === 'student') {
          lists[person.grade].push({
            name: person.name,
            tableNumber: tableIndex + 1
          })
        }
      })
    })
  } else {
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
  }

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
  document.body.classList.add('print-table-cards')
  setTimeout(() => {
    window.print()
    setTimeout(() => {
      printMode.value = false
      document.body.classList.remove('print-table-cards')
    }, 500)
  }, 500)
}

const printStudentLists = () => {
  printMode.value = true
  setTimeout(() => {
    window.print()
    printMode.value = false
  }, 100)
}

const rotationItems = computed(() => 
  Array.from({length: numberOfRotations.value}, (_, i) => ({
    title: `Rotation ${i + 1}`,
    value: i + 1
  }))
)
</script>

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
  margin-bottom: 2rem;
  flex-wrap: wrap;
}
.tabs .v-btn {
  margin-right: 8px;
  margin-bottom: 8px;
}

.grade-section {
  margin-bottom: 3rem;
  padding: 1rem;
  background-color: #f9f9f9;
  border-radius: 4px;
}

.tables-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}

.table-card {
  border: 1px solid #ccc;
  border-radius: 4px;
  padding: 1.5rem;
  margin-bottom: 1rem;
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
  gap: 20px;
}

.table-card-print {
  width: 7.5in;
  height: 4.5in;
  border: 1px solid #000;
  margin: 0;
  padding: 0.25in;
  box-sizing: border-box;
  page-break-inside: avoid;
  display: flex;
  flex-direction: column;
  position: relative;
  overflow: hidden;
}

.table-card-print::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-image: url('/logo.png'); /* Ensure this path is correct */
  background-size: contain;
  background-repeat: no-repeat;
  background-position: center;
  opacity: 0.2;
  pointer-events: none;
}

.table-number {
  font-size: 24px;
  text-align: center;
  margin-bottom: 1rem;
}

.grades-row {
  display: flex;
  justify-content: center;
  flex-grow: 1;
  gap: 1rem;
  width: 100%;
}

.grade-list {
  flex: 0 1 30%; /* Don't grow, allow shrinking, 30% basis */
  max-width: 30%; /* Maximum width of 30% */
  min-width: 0; /* Allow shrinking below content size */
}

/* Center single grade */
.grades-row:only-child {
  justify-content: center;
}

.grades-row:only-child .grade-list {
  flex-basis: 60%;
  max-width: 60%;
}

/* Center two grades */
.grades-row:nth-last-child(2):first-child {
  justify-content: space-around;
}

.grades-row:nth-last-child(2):first-child .grade-list {
  flex-basis: 45%;
  max-width: 45%;
}

.grade-list:first-child {
  border-left: none;
  padding-left: 0;
}

.grade-list h5 {
  font-size: 18px;
  margin-top: 0;
  margin-bottom: 0.25rem;
  text-align: center;
}

.grade-list ul {
  margin: 0;
  padding-left: 1rem; /* Add some left padding for the list */
  list-style-type: none; /* Remove default list styling */
}

.grade-list li {
  font-size: 16px;
  margin-bottom: 0.15rem;
}

@media print {
  @page {
    size: letter portrait;
    margin: 0;
  }

  body {
    margin: 0;
    padding: 0;
  }

  .table-cards-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 0;
  }

  .table-card-print {
    width: 7.5in;
    height: 5in;
    border: 1px solid #000;
    padding: 0.25in;
    margin: 0.5in 0.5in 0 0.5in; /* Top Right Bottom Left */
    box-sizing: border-box;
    page-break-inside: avoid;
    page-break-after: always;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .table-card-print:nth-child(even) {
    margin-top: 0;
  }

  .table-card-print:last-child {
    page-break-after: auto;
  }

  .table-number {
    font-size: 24px;
    text-align: center;
    margin-bottom: 0.3rem;
  }

  .grades-row {
    display: flex;
    justify-content: space-around;
    flex-grow: 1;
  }

  .grade-list {
    flex: 1;
    max-width: 30%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
  }

  .grade-list h5 {
    font-size: 18px;
    margin: 0 0 0.2rem 0;
    text-align: center;
  }

  .grade-list ul {
    list-style-type: none;
    padding: 0;
    margin: 0;
    flex-grow: 1;
    overflow-y: auto;
  }

  .grade-list li {
    font-size: 12px;
    margin-bottom: 0.1rem;
  }

  .v-application {
    background: none !important;
  }

  .v-main {
    padding: 0 !important;
  }

  .v-container {
    padding: 0 !important;
    max-width: none !important;
  }

  .no-print, .v-btn, h1, h2 {
    display: none !important;
  }

  .print-content {
    visibility: visible;
    position: static;
    left: auto;
    top: auto;
    width: auto;
    height: auto;
    padding: 0;
  }

  .table-cards-container {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    gap: 0.8cm; /* Add gap between cards */
  }

  .table-card-print {
    page-break-inside: avoid;
    margin: 0;
    padding: 0.25in;
    border: 1px solid #000;
    height: calc(5in - 0.4cm); /* Adjust height to account for gap */
    width: 7.5in;
    box-sizing: border-box;
  }

  .table-card-print:nth-child(odd) {
    page-break-after: avoid;
  }

  .table-card-print:nth-child(even) {
    page-break-after: always;
  }

  /* Remove these pseudo-elements */
  /*
  .table-cards-container::before,
  .table-cards-container::after {
    content: '';
    display: block;
    height: 0.4cm;
    page-break-before: always;
  }
  */

  .student-lists-container {
    display: flex;
    font-size: 16pt; /* Even smaller font for print */
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
    font-size: 20pt; /* Smaller heading for print */
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

  .grade-list {
    flex: 0 1 30%;
    max-width: 30%;
  }

  /* Center single grade in print */
  .grades-row:only-child .grade-list {
    flex-basis: 60%;
    max-width: 60%;
  }

  /* Center two grades in print */
  .grades-row:nth-last-child(2):first-child .grade-list {
    flex-basis: 45%;
    max-width: 45%;
  }

  .grade-list li {
    font-size: 16px;
  }
}

.student-lists-container {
  display: flex;
  font-size: 16px; /* Reduced font size */
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
  font-size: 18px; /* Reduced heading size */
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
  max-width: 200px;
  margin-bottom: 2rem;
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

.v-main {
  background-color: #f5f5f5; /* Light grey background for the entire app */
}

.v-container {
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* You may need to adjust other styles to fit Vuetify's design */

.table-card-note {
  text-align: center;
  font-style: italic;
  margin-bottom: 0.5rem;
}

@media print {
  /* ... existing print styles ... */

  .table-card-note {
    font-size: 14px;
    margin-bottom: 0.3rem;
  }

  /* ... rest of the print styles ... */
}

.people-list {
  list-style-type: none;
  padding: 0;
  margin: 0;
  flex-grow: 1;
  overflow-y: auto;
}

.people-list li {
  font-size: 14px;
  margin-bottom: 0.2rem;
}

@media print {
  /* ... existing print styles ... */

  .people-list li {
    font-size: 12px;
    margin-bottom: 0.1rem;
  }
}
</style>





