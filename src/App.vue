<script setup>
import { ref, onMounted } from 'vue'

// Constants
const CANVAS_SIZE = 750
const SHAPE_SCALE = 600
const SHAPE_PADDING = 75
const DIMENSION_OFFSET = 50
const SHAPE_LINE_WIDTH = 2
const DIMENSION_LINE_WIDTH = 3

const shapeFiles = ['simple.json', 't_shape.json', 'triangle.json']

// Refs for canvas and context
const canvas = ref(null)
const ctx = ref(null)
const currentCorners = ref([])
const perpendicularOptions = ref([])
const currentDimensionIndex = ref(0)

//Function to fetch shape data from JSON files
async function fetchShape(filename) {
  const response = await fetch(`/${filename}`)
  return await response.json()
}

// Function to normalize coordinates for drawing on canvas
function normalizeCoordinates(corners) {
  const xs = corners.map((c) => c.x)
  const ys = corners.map((c) => c.y)

  const minX = Math.min(...xs)
  const maxX = Math.max(...xs)
  const minY = Math.min(...ys)
  const maxY = Math.max(...ys)

  const width = maxX - minX
  const height = maxY - minY
  const scale = Math.min(SHAPE_SCALE / width, SHAPE_SCALE / height)

  return corners.map((corner) => ({
    x: (corner.x - minX) * scale + SHAPE_PADDING,
    y: (corner.y - minY) * scale + SHAPE_PADDING,
  }))
}

// Function to trace the perimeter of a shape based on its corners
function traceShapePerimeter(corners) {
  if (corners.length <= 2) return corners

  // Build connections between corners using walls
  const connections = {}
  corners.forEach((corner) => {
    connections[corner.id] = []
  })

  // Connect corners that share walls
  corners.forEach((corner) => {
    corner.wallStarts?.forEach((wall) => {
      const connectedCorner = corners.find((c) => c.wallEnds?.some((w) => w.id === wall.id))
      if (connectedCorner) {
        connections[corner.id].push(connectedCorner)
      }
    })
  })

  // Start from any corner and follow connections
  const visited = []
  const path = []
  let current = corners[0]

  // Loop through corners to trace the perimeter
  while (current && !visited.includes(current.id)) {
    path.push(current)
    visited.push(current.id)

    // Find next unvisited connected corner
    const neighbors = connections[current.id] || []
    current = neighbors.find((neighbor) => !visited.includes(neighbor.id))
  }

  return path
}

// Function to draw the canvas with current or new shape data
function drawCanvas(newCorners = null) {
  const context = ctx.value
  if (!context) return

  // If new corners provided, update state
  if (newCorners) {
    currentCorners.value = newCorners
    perpendicularOptions.value = identifyPerpendicularOptions(newCorners)
    currentDimensionIndex.value = 0 // Reset to first perpendicular option
  }

  // Exit if no corners available
  if (currentCorners.value.length === 0) return

  // Clear canvas
  context.clearRect(0, 0, CANVAS_SIZE, CANVAS_SIZE)

  // Draw shape outline
  let orderedCorners = traceShapePerimeter(currentCorners.value)
  const normalizedCorners = normalizeCoordinates(orderedCorners)

  context.beginPath()
  context.moveTo(normalizedCorners[0].x, normalizedCorners[0].y)
  for (let i = 1; i < normalizedCorners.length; i++) {
    context.lineTo(normalizedCorners[i].x, normalizedCorners[i].y)
  }
  context.closePath()
  context.strokeStyle = '#000'
  context.lineWidth = SHAPE_LINE_WIDTH
  context.stroke()
  context.fillStyle = 'rgba(300, 500, 0, 0.1)'
  context.fill()

  // Draw dimensions if available
  if (perpendicularOptions.value.length > 0) {
    const currentOption = perpendicularOptions.value[currentDimensionIndex.value]
    if (currentOption) {
      drawDimensionLines(currentOption, normalizedCorners)
    }
  }
}

// Function to load a random shape from the list of shapes
async function loadRandomShape() {
  const shapeData = await fetchShape(shapeFiles[Math.floor(Math.random() * shapeFiles.length)])
  drawCanvas(shapeData.corners)
}

// Function to check if two vectors are perpendicular using dot product of two vectors
function isPerpendicular(v1, v2) {
  const dotProduct = v1[0] * v2[0] + v1[1] * v2[1]
  return Math.abs(dotProduct) === 0
}

// Function to find perpendicular wall pairs in the shape
function findPerpendicularWalls(corners) {
  const walls = []

  // Extract all walls as vectors
  for (let i = 0; i < corners.length; i++) {
    const current = corners[i]
    const next = corners[(i + 1) % corners.length]
    walls.push([next.x - current.x, next.y - current.y])
  }

  // Find perpendicular pairs
  const perpendicularPairs = []
  for (let i = 0; i < walls.length; i++) {
    for (let j = i + 1; j < walls.length; j++) {
      if (isPerpendicular(walls[i], walls[j])) {
        perpendicularPairs.push([i, j])
      }
    }
  }

  return perpendicularPairs
}

// Function to identify all possible perpendicular options (perpendicular wall orientations)
function identifyPerpendicularOptions(corners) {
  const perpendicularPairs = findPerpendicularWalls(corners)

  if (perpendicularPairs.length === 0) return []

  // For shapes with perpendicular walls, offer both orientations
  return [{ isLengthHorizontal: true }, { isLengthHorizontal: false }]
}

// Function to draw dimension lines on canvas
function drawDimensionLines(pair, normalizedCorners) {
  const context = ctx.value
  if (!context) return

  // Get shape boundaries from already normalized coordinates
  const minX = Math.min(...normalizedCorners.map((c) => c.x))
  const maxX = Math.max(...normalizedCorners.map((c) => c.x))
  const minY = Math.min(...normalizedCorners.map((c) => c.y))
  const maxY = Math.max(...normalizedCorners.map((c) => c.y))

  // Determine line positions based on interpretation
  let lengthLine, widthLine

  if (pair.isLengthHorizontal) {
    lengthLine = {
      start: { x: minX, y: maxY + DIMENSION_OFFSET },
      end: { x: maxX, y: maxY + DIMENSION_OFFSET },
    }
    widthLine = {
      start: { x: minX - DIMENSION_OFFSET, y: minY },
      end: { x: minX - DIMENSION_OFFSET, y: maxY },
    }
  } else {
    lengthLine = {
      start: { x: minX - DIMENSION_OFFSET, y: minY },
      end: { x: minX - DIMENSION_OFFSET, y: maxY },
    }
    widthLine = {
      start: { x: minX, y: maxY + DIMENSION_OFFSET },
      end: { x: maxX, y: maxY + DIMENSION_OFFSET },
    }
  }

  // Helper function to draw a line
  const drawLine = (line, color) => {
    context.strokeStyle = color
    context.lineWidth = DIMENSION_LINE_WIDTH
    context.beginPath()
    context.moveTo(line.start.x, line.start.y)
    context.lineTo(line.end.x, line.end.y)
    context.stroke()
  }

  // Draw lines
  drawLine(lengthLine, '#FF0000') // Red LENGTH line
  drawLine(widthLine, '#0000FF') // Blue WIDTH line
}

// Function to cycle through different dimension options
function cycleDimensions() {
  if (perpendicularOptions.value.length === 0) return

  // Simple toggle between 0 and 1 (since there are only 2 options)
  currentDimensionIndex.value = currentDimensionIndex.value === 0 ? 1 : 0

  drawCanvas() // Redraw with current data
}

// Using the onMounted lifecycle hook to initialize the canvas and load a random shape
onMounted(async () => {
  const canvasElement = canvas.value
  if (canvasElement) {
    ctx.value = canvasElement.getContext('2d')
    await loadRandomShape()
  }
})
</script>

<template>
  <div
    style="
      padding: 20px;
      display: flex;
      flex-direction: column;
      align-items: center;
      min-height: 100vh;
      justify-content: center;
    "
  >
    <div style="display: flex; gap: 10px; margin-bottom: 20px">
      <button
        @click="loadRandomShape"
        style="
          padding: 10px 20px;
          font-size: 16px;
          background-color: #e4df00;
          color: black;
          border: none;
          border-radius: 5px;
          cursor: pointer;
        "
      >
        Load Random Shape
      </button>

      <button
        @click="cycleDimensions"
        style="
          padding: 10px 20px;
          font-size: 16px;
          background-color: #1a1905;
          color: white;
          border: none;
          border-radius: 5px;
          cursor: pointer;
        "
      >
        Next Length/Width Option
      </button>
    </div>

    <canvas
      ref="canvas"
      :width="CANVAS_SIZE"
      :height="CANVAS_SIZE"
      style="border: 1px solid #000; border-radius: 5px"
    ></canvas>

    <div v-if="perpendicularOptions.length > 0" style="margin-top: 20px; text-align: center">
      <p>
        <span style="color: #ff0000; font-weight: bold">Red Line = LENGTH</span> |
        <span style="color: #0000ff; font-weight: bold">Blue Line = WIDTH</span>
      </p>
    </div>
    <div v-else style="margin-top: 20px; text-align: center">
      <p>No perpendicular options available for this shape</p>
    </div>
  </div>
</template>

<style scoped></style>
