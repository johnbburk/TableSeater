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
                  placeholder="Last Name, First Name&#9;9"
                ></v-textarea>
              </v-card-text>
              <v-card-actions>
                <v-btn @click="importPeople('student')" color="primary">Import Students</v-btn>
                <v-btn @click="loadDemoStudents" color="secondary">Load Demo Students</v-btn>
                <v-btn @click="resetList('student')" color="error">Reset Students</v-btn>
              </v-card-actions>
            </v-card>

            <v-card>
              <v-card-text>
                <v-textarea
                  v-model="teacherInput"
                  label="Teachers (name and optional table number, tab-separated, one per line)"
                  rows="5"
                  placeholder="Last Name, First Name&#9;10&#9;1"
                ></v-textarea>
              </v-card-text>
              <v-card-actions>
                <v-btn @click="importPeople('teacher')" color="primary">Import Teachers</v-btn>
                <v-btn @click="loadDemoTeachers" color="secondary">Load Demo Teachers</v-btn>
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
              <!-- Whole school seating -->
              <template v-if="seatingMode === 'whole'">
                <div v-for="card in tableCards" :key="card.tableNumber" class="table-card-print">
                  <h3 class="table-number">Table {{ card.tableNumber }}</h3>
                  <p v-if="tableCardNote" class="table-card-note">{{ tableCardNote }}</p>
                  <ul class="people-list single-column-list">
                    <li v-for="person in (card as WholeSchoolCard).people" :key="person.name">
                      <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                      <span v-else>{{ person.name }} (Grade {{ person.grade }})</span>
                    </li>
                  </ul>
                </div>
              </template>

              <!-- Individual grade seating -->
              <template v-else>
                <div v-for="card in tableCards" :key="card.tableNumber" class="table-card-print">
                  <h3 class="table-number">Table {{ card.tableNumber }}</h3>
                  <p v-if="tableCardNote" class="table-card-note">{{ tableCardNote }}</p>
                  <ul class="people-list grade-columns">
                    <template v-for="grade in [6, 7, 8]" :key="grade">
                      <div v-if="(card as GradeCard).gradeGroups[grade]?.length > 0" class="grade-column">
                        <h3 class="grade-header">Grade {{ grade }}</h3>
                        <li v-for="person in (card as GradeCard).gradeGroups[grade]" :key="person.name">
                          <strong v-if="person.type === 'teacher'">{{ person.name }}</strong>
                          <span v-else>{{ person.name }}</span>
                        </li>
                      </div>
                    </template>
                  </ul>
                </div>
              </template>
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
  grade?: number
  tableNumber?: number // Add optional tableNumber property
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
const tablesPerGrade = ref<GradeGroups>({
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
    teacherInput.value = teachers.value.map(t => `${t.name}\t${t.grade}${t.tableNumber ? '\t' + t.tableNumber : ''}`).join('\n')
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

// Add this function to load demo students
const loadDemoStudents = async () => {
  try {
    const response = await fetch('demoStudents.csv')
    const text = await response.text()
    studentInput.value = text
  } catch (error) {
    console.error('Error loading demo students:', error)
  }
}

// Modify the importPeople function
const importPeople = async (type: 'student' | 'teacher') => {
  const input = type === 'student' ? studentInput.value : teacherInput.value
  
  const people = input
    .split('\n')
    .filter(line => line.trim() !== '')
    .map(line => {
      const parts = line.split('\t')
      const person: Person = {
        name: parts[0].trim(),
        type,
        grade: type === 'student' ? parseInt(parts[1]?.trim() || '0', 10) : undefined
      }
      
      // Add tableNumber for teachers if provided
      if (type === 'teacher' && parts.length > 1) {
        person.tableNumber = parseInt(parts[1]?.trim() || '0', 10)
      }
      
      return person
    })

  if (type === 'student') {
    students.value = people
    studentInput.value = students.value.map(s => `${s.name}\t${s.grade}`).join('\n')
  } else {
    teachers.value = people
    teacherInput.value = teachers.value.map(t => `${t.name}\t${t.grade}${t.tableNumber ? '\t' + t.tableNumber : ''}`).join('\n')
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
      const allTeachers = [...teachers.value]
      
      // Create arrays for each grade and randomize them
      const grade8Students = [...students.value.filter(s => s.grade === 8)].sort(() => Math.random() - 0.5)
      const grade7Students = [...students.value.filter(s => s.grade === 7)].sort(() => Math.random() - 0.5)
      const grade6Students = [...students.value.filter(s => s.grade === 6)].sort(() => Math.random() - 0.5)
      
      allTables[0] = Array.from({ length: totalTables.value }, () => [])

      // Distribute teachers based on tableNumber if specified
      allTeachers.forEach((teacher) => {
        if (teacher.tableNumber && teacher.tableNumber <= totalTables.value) {
          allTables[0][teacher.tableNumber - 1].push(teacher)
        } else {
          const index = allTables[0].findIndex(table => !table.some(p => p.type === 'teacher'))
          if (index >= 0) {
            allTables[0][index].push(teacher)
          }
        }
      })

      // Distribute students by grade (8th, then 7th, then 6th)
      grade8Students.forEach((student, index) => {
        allTables[0][index % totalTables.value].push(student)
      })
      
      grade7Students.forEach((student, index) => {
        allTables[0][index % totalTables.value].push(student)
      })
      
      grade6Students.forEach((student, index) => {
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

        // Distribute teachers based on tableNumber if specified
        gradeTeachers.forEach((teacher) => {
          if (teacher.tableNumber && teacher.tableNumber <= numberOfTables) {
            gradeTables[teacher.tableNumber - 1].push(teacher)
          } else {
            const index = gradeTables.findIndex(table => !table.some(p => p.type === 'teacher'))
            if (index >= 0) {
              gradeTables[index].push(teacher)
            }
          }
        })

        // Randomly distribute students
        const shuffledStudents = [...gradeStudents].sort(() => Math.random() - 0.5)
        shuffledStudents.forEach((student, index) => {
          gradeTables[index % numberOfTables].push(student)
        })

        // Sort people within each table: teachers first, then students remain in random order
        allTables[grade] = gradeTables.map(table => {
          const teachers = table.filter(person => person.type === 'teacher')
          const students = table.filter(person => person.type === 'student')
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

// Add these interfaces at the top of the script section
interface GradeGroups {
  6: Person[];
  7: Person[];
  8: Person[];
}

interface WholeSchoolCard {
  type: 'whole';
  tableNumber: number;
  people: Person[];
}

interface GradeCard {
  type: 'individual';
  tableNumber: number;
  gradeGroups: GradeGroups;
}

type TableCard = WholeSchoolCard | GradeCard;

// Update the tableCards computed property
const tableCards = computed((): TableCard[] => {
  if (seatingMode.value === 'whole') {
    return Array.from({ length: totalTables.value }, (_, index) => {
      const tableNumber = index + 1
      const people = tables.value[currentRotation.value]?.[0]?.[index] || []
      
      return {
        type: 'whole' as const,
        tableNumber,
        people: [...people].sort((a, b) => {
          if (a.type !== b.type) return a.type === 'teacher' ? -1 : 1
          if (a.grade && b.grade && a.grade !== b.grade) return b.grade - a.grade
          return a.name.localeCompare(b.name)
        })
      }
    }).filter(card => card.people.length > 0)
  } else {
    return Array.from({ length: Math.max(...Object.values(tablesPerGrade.value)) }, (_, index) => {
      const tableNumber = index + 1
      return {
        type: 'individual' as const,
        tableNumber,
        gradeGroups: {
          6: tables.value[currentRotation.value]?.[6]?.[index] || [],
          7: tables.value[currentRotation.value]?.[7]?.[index] || [],
          8: tables.value[currentRotation.value]?.[8]?.[index] || []
        }
      }
    }).filter(card => Object.values(card.gradeGroups).some(group => group.length > 0))
  }
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
  const printWindow = window.open('', '_blank')
  if (!printWindow) return

  const rotation = currentRotation.value
  const logoUrl = new URL('/logo.png', window.location.href).href

  const content = `
    <!DOCTYPE html>
    <html>
    <head>
      <title>Table Cards - Rotation ${rotation}</title>
      <style>
        @page {
          size: letter portrait;
          margin: 1in 0;
        }
        body {
          margin: 0;
          padding: 0;
          font-family: Arial, sans-serif;
          display: flex;
          flex-wrap: wrap;
          justify-content: center;
          column-gap: 0;
          row-gap: 0.5in;
        }
        .table-card {
          width: 6.5in;
          height: 4.25in;
          padding: 0.25in;
          position: relative;
          box-sizing: border-box;
          border: 1px solid #000;
          background-color: white;
          break-inside: avoid;
          page-break-inside: avoid;
          display: flex;
          flex-direction: column;
          align-items: center;
        }
        .table-card:nth-child(2n) {
          page-break-after: always;
        }
        .table-card::before {
          content: '';
          position: absolute;
          top: 0;
          left: 0;
          right: 0;
          bottom: 0;
          background-image: url("${logoUrl}");
          background-size: 60% auto;
          background-repeat: no-repeat;
          background-position: center;
          opacity: 0.15;
          pointer-events: none;
          z-index: 0;
          -webkit-print-color-adjust: exact;
          print-color-adjust: exact;
        }
        .table-number {
          font-size: 24px;
          text-align: center;
          margin-bottom: 0.15in;
          position: relative;
          z-index: 1;
          font-weight: bold;
        }
        .table-note {
          text-align: center;
          font-style: italic;
          margin-bottom: 0.15in;
          position: relative;
          z-index: 1;
          font-size: 14px;
        }
        .grade-columns {
          display: flex;
          justify-content: space-between;
          width: 95%;
          margin: 0 auto;
          gap: 0.25in;
          position: relative;
          z-index: 1;
        }
        .grade-column {
          flex: 1;
          text-align: left;
        }
        .grade-header {
          font-size: 16px;
          margin-bottom: 0.1in;
          text-align: center;
          font-weight: bold;
        }
        .people-list {
          list-style: none;
          padding: 0;
          margin: 0;
          font-size: 14px;
          line-height: 1.4;
        }
        .people-list li {
          margin-bottom: 0.08in;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
        }
        .single-column-list {
          width: 80%;
          margin: 0 auto;
          text-align: center;
          display: flex;
          flex-direction: column;
          align-items: center;
        }
        .single-column-list li {
          font-size: 13px;
          margin-bottom: 0.08in;
          white-space: nowrap;
          overflow: hidden;
          text-overflow: ellipsis;
          width: 100%;
        }
        .single-column-list li strong {
          font-size: 15px;
          margin-bottom: 0.08in;
          display: block;
          width: 100%;
        }
        .rotation-date {
          font-size: 16px;
          text-align: center;
          margin-bottom: 0.15in;
          font-style: italic;
          position: relative;
          z-index: 1;
        }
      </style>
    </head>
    <body>
      ${tableCards.value.map(card => {
        if (card.type === 'whole') {
          return `
            <div class="table-card">
              <h3 class="table-number">Table ${card.tableNumber}</h3>
              ${tableCardNote.value ? `<p class="table-note">${tableCardNote.value}</p>` : ''}
              <ul class="people-list single-column-list">
                ${card.people.map(person => `
                  <li>
                    ${person.type === 'teacher' 
                      ? `<strong>${person.name}</strong>` 
                      : `${person.name} (Grade ${person.grade})`}
                  </li>
                `).join('')}
              </ul>
            </div>
          `
        } else {
          return `
            <div class="table-card">
              <h3 class="table-number">Table ${card.tableNumber}</h3>
              ${tableCardNote.value ? `<p class="table-note">${tableCardNote.value}</p>` : ''}
              <div class="grade-columns">
                ${[6, 7, 8].map(grade => {
                  const people = card.gradeGroups[grade as keyof GradeGroups]
                  if (people.length === 0) return ''
                  return `
                    <div class="grade-column">
                      <h3 class="grade-header">Grade ${grade}</h3>
                      <ul class="people-list">
                        ${people.map(person => `
                          <li>${person.type === 'teacher' ? `<strong>${person.name}</strong>` : 
                            person.name}</li>
                        `).join('')}
                      </ul>
                    </div>
                  `
                }).join('')}
              </div>
            </div>
          `
        }
      }).join('')}
    </body>
    </html>
  `

  printWindow.document.write(content)
  printWindow.document.close()

  printWindow.onload = () => {
    const img = new Image()
    img.src = logoUrl
    img.onload = () => {
      setTimeout(() => {
        printWindow.print()
        setTimeout(() => {
          printWindow.close()
        }, 1000)
      }, 500)
    }
    img.onerror = () => {
      console.error('Failed to load logo image')
      setTimeout(() => {
        printWindow.print()
        setTimeout(() => {
          printWindow.close()
        }, 1000)
      }, 500)
    }
  }
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

const loadDemoTeachers = async () => {
  try {
    const response = await fetch('demoTeachers.csv')
    const text = await response.text()
    teacherInput.value = text
  } catch (error) {
    console.error('Error loading demo teachers:', error)
  }
}
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
  flex-direction: column;
  gap: 20px;
  padding: 20px;
  align-items: center;
}

.table-card-print {
  width: 6.5in;
  height: 4.25in;
  border: 1px solid #000;
  padding: 0.25in;
  margin-bottom: 1rem;
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  box-sizing: border-box;
  flex: none;
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
  opacity: 0.2;
  pointer-events: none;
  z-index: 0;
}

.table-number, .table-card-note, .people-list {
  position: relative; /* Place above background */
  z-index: 1;
}

/* Style for whole school seating */
.table-card-print .people-list:not(.grade-columns) {
  list-style-type: none;
  padding: 0;
  margin: 0 auto; /* Center the list */
  width: calc(33.33% - 1.33rem); /* Match the width of grade columns */
  text-align: left;
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
    padding: 0;
    margin: 0;
  }

  .table-card-print {
    width: 6.5in;
    height: 4.25in;
    margin: 0 auto;
    padding: 0.25in;
    page-break-after: always;
    break-inside: avoid;
    box-sizing: border-box;
  }

  /* Remove any margin from the last card */
  .table-card-print:last-child {
    page-break-after: auto;
  }

  /* Center cards on page */
  .print-content {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.5in;
  }

  /* Ensure content fits within card dimensions */
  .people-list.grade-columns {
    max-height: 3in; /* Account for padding and headers */
    overflow: hidden;
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
  font-size: 20px;
  margin-bottom: 0.2rem;
}

@media print {
  /* ... existing print styles ... */

  .people-list li {
    font-size: 12px;
    margin-bottom: 0.1rem;
  }
}

.people-list.grade-columns {
  display: flex;
  justify-content: space-between;
  width: 95%;
  margin: 0 auto;
  padding: 0;
  gap: 0.25in;
}

.people-list.grade-columns .grade-column {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  padding: 0 0.1in;
}

.grade-header {
  font-size: 16px;
  margin-bottom: 0.2in;
  width: 100%;
  text-align: center;
}

.people-list.grade-columns li {
  width: 100%;
  padding: 0.1rem 0;
  word-wrap: break-word;
  overflow-wrap: break-word;
  font-size: 11px;
  line-height: 1.2;
}

@media print {
  @page {
    size: letter portrait;
    margin: 0;
  }

  .table-cards-container {
    display: flex;
    flex-direction: column;
    gap: 0;
    justify-content: center;
  }

  .table-card-print {
    width: 6.5in;
    height: 4.25in;
    margin: 0 auto;
    margin-bottom: 0.25in;
    page-break-after: always;
    break-inside: avoid;
  }

  .table-card-print:nth-child(odd) {
    margin-right: auto;
  }

  .people-list.grade-columns {
    width: 95%;
    gap: 0.25in;
  }

  .people-list.grade-columns .grade-column {
    flex: 1;
    padding: 0 0.1in;
  }

  .grade-header {
    font-size: 14px;
    margin-bottom: 0.15in;
  }

  .people-list.grade-columns li {
    font-size: 10px;
    line-height: 1.2;
    margin-bottom: 0.05in;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  .table-card-print::before {
    background-size: 60% auto;
    opacity: 0.15;
  }

  .table-card-print {
    height: 4.25in !important;
    width: 6.5in !important;
  }
}

/* Base styles for print/no-print */
.no-print {
  display: block;
}

.print-content {
  display: block;
}

@media print {
  /* Hide elements with no-print class */
  .no-print {
    display: none !important;
  }

  /* Show the print content */
  .print-content {
    display: block !important;
  }

  /* Ensure table cards container is visible */
  .table-cards-container {
    display: flex !important;
    flex-direction: column;
    padding: 0;
    margin: 0;
    visibility: visible !important;
  }

  /* Ensure table cards are visible */
  .table-card-print {
    display: flex !important;
    width: 6.5in;
    height: 4.25in;
    margin: 0 auto;
    padding: 0.25in;
    page-break-after: always;
    break-inside: avoid;
    box-sizing: border-box;
    visibility: visible !important;
  }

  /* Hide all tabs except the active table cards tab */
  .print-content > div:not([v-if="activeTab === 'cards'"]) {
    display: none !important;
  }

  /* Remove any margins/padding that might affect layout */
  body {
    margin: 0;
    padding: 0;
  }

  .v-container {
    padding: 0 !important;
    margin: 0 !important;
  }

  .v-main {
    padding: 0 !important;
  }

  /* Ensure content within cards is visible */
  .table-number,
  .table-card-note,
  .people-list,
  .grade-column,
  .grade-header {
    visibility: visible !important;
    display: block !important;
  }

  /* Keep background image visible */
  .table-card-print::before {
    visibility: visible !important;
  }
}
</style>





