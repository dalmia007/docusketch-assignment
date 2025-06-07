<script setup>
import { ref, onMounted } from 'vue'
const shapeFiles = ['simple.json', 't_shape.json', 'triangle.json']

// Refs for canvas and context
const canvas = ref(null)
const ctx = ref(null)

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
  const scale = Math.min(600 / width, 600 / height)

  return corners.map((corner) => ({
    x: (corner.x - minX) * scale + 75,
    y: (corner.y - minY) * scale + 75,
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

// Function to draw the shape on the canvas
function drawShape(shapeData) {
  const context = ctx.value
  if (!context) return

  // Clear canvas
  context.clearRect(0, 0, 750, 750)

  // Get corners from shape data
  const corners = shapeData.corners

  // Trace the proper perimeter as corners are not always ordered
  let orderedCorners = traceShapePerimeter(corners)

  // Normalize coordinates to fit canvas
  const normalizedCorners = normalizeCoordinates(orderedCorners)

  // Draw the shape
  context.beginPath()
  context.moveTo(normalizedCorners[0].x, normalizedCorners[0].y)

  for (let i = 1; i < normalizedCorners.length; i++) {
    context.lineTo(normalizedCorners[i].x, normalizedCorners[i].y)
  }

  context.closePath()
  context.strokeStyle = '#000'
  context.lineWidth = 2
  context.stroke()
  context.fillStyle = 'rgba(300, 500, 0, 0.1)'
  context.fill()
}

// Function to load a random shape from the list of shapes
async function loadRandomShape() {
  const shapeData = await fetchShape(shapeFiles[Math.floor(Math.random() * shapeFiles.length)])
  drawShape(shapeData)
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
    <canvas ref="canvas" width="750" height="750" style="border: 1px solid #000"></canvas>
  </div>
</template>

<style scoped></style>
