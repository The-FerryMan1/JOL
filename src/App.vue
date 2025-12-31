<script setup lang="ts">
import {
  Engine,
  Bodies,
  Runner,
  Composite,
  Render,
  Mouse,
  MouseConstraint,
  Events,
  Query,
  Body,
} from 'matter-js'
import { onMounted, onUnmounted, useTemplateRef, ref } from 'vue'

// --- VUE STATE ---
const canvasContainer = useTemplateRef<HTMLElement>('canvas')
const selectedNote = ref<string | null>(null)

// --- MATTER-JS INSTANCES ---
let engine: Engine
let render: Render
let runner: Runner
let mouseConstraint: MouseConstraint

// Physics Bodies
let wallLeft: Body | undefined
let wallRight: Body | undefined
let floor: Body | undefined
let ceiling: Body | undefined

const messages = [
  'Ikaw ang pinakamagandang babae na nakilala!',
  'Nami-miss ko na ang imong kiss, Love!',
  'Sorry for everything.',
  '.',
  "I'm so proud of all you do.",
]

onMounted(() => {
  if (!canvasContainer.value) return

  engine = Engine.create()

  render = Render.create({
    element: canvasContainer.value,
    engine: engine,
    options: {
      width: window.innerWidth,
      height: window.innerHeight,
      wireframes: false,
      background: 'transparent',
      pixelRatio: 1,
    },
  })

  const thickness = 100
  const w = window.innerWidth
  const h = window.innerHeight

  // --- 1. THE BOX ---
  floor = Bodies.rectangle(w / 2, h + thickness / 2, w * 5, thickness, {
    isStatic: true,
    label: 'ground',
  })
  ceiling = Bodies.rectangle(w / 2, -thickness / 2, w * 5, thickness, { isStatic: true })
  wallLeft = Bodies.rectangle(-thickness / 2, h / 2, thickness, h * 5, { isStatic: true })
  wallRight = Bodies.rectangle(w + thickness / 2, h / 2, thickness, h * 5, { isStatic: true })

  // --- 2. THE NOTES ---
  const notes = messages.map((text, i) => {
    return Bodies.rectangle(
      w / 2 + i * 10,
      h - 100 - i * 20, // Start them near the bottom so they don't drop far
      140,
      40,
      {
        label: text,
        chamfer: { radius: 12 },
        render: {
          fillStyle: ['#fb7185', '#f472b6', '#f9a8d4', '#fda4af'][i % 4],
          strokeStyle: '#ffffff',
          lineWidth: 3,
        },
      },
    )
  })

  Composite.add(engine.world, [floor, ceiling, wallLeft, wallRight, ...notes])

  // --- 3. MOUSE INTERACTION ---
  const mouse = Mouse.create(render.canvas)
  render.mouse = mouse

  mouseConstraint = MouseConstraint.create(engine, {
    mouse: mouse,
    constraint: { stiffness: 0.2, render: { visible: false } },
  })

  let startPos = { x: 0, y: 0 }
  Events.on(mouseConstraint, 'mousedown', (event) => {
    startPos = { x: event.mouse.position.x, y: event.mouse.position.y }
  })

  Events.on(mouseConstraint, 'mouseup', (event) => {
    const endPos = event.mouse.position
    const distance = Math.sqrt(
      Math.pow(endPos.x - startPos.x, 2) + Math.pow(endPos.y - startPos.y, 2),
    )

    if (distance < 10) {
      let clickedBody = mouseConstraint.body
      if (!clickedBody) {
        const bodiesUnderMouse = Query.point(notes, endPos)
        if (bodiesUnderMouse.length > 0) clickedBody = bodiesUnderMouse[0] as Body
      }
      if (clickedBody && messages.includes(clickedBody.label)) {
        selectedNote.value = clickedBody.label
      }
    }
  })

  Composite.add(engine.world, mouseConstraint)

  Render.run(render)
  runner = Runner.create()
  Runner.run(runner, engine)

  // --- 4. IMPROVED RESPONSIVE RESIZE ---
  const handleResize = () => {
    const newW = window.innerWidth
    const newH = window.innerHeight

    render.canvas.width = newW
    render.canvas.height = newH
    render.options.width = newW
    render.options.height = newH

    // We use Body.setPosition to move the walls.
    // To prevent notes from falling, we check if they are below the new floor
    if (floor && ceiling && wallLeft && wallRight) {
      const oldFloorY = floor.position.y
      const newFloorY = newH + thickness / 2

      Body.setPosition(floor, { x: newW / 2, y: newFloorY })
      Body.setPosition(ceiling, { x: newW / 2, y: -thickness / 2 })
      Body.setPosition(wallLeft, { x: -thickness / 2, y: newH / 2 })
      Body.setPosition(wallRight, { x: newW + thickness / 2, y: newH / 2 })

      // If the window got smaller (pushed floor UP),
      // we need to nudge notes up so they don't get buried in the floor
      const allBodies = Composite.allBodies(engine.world)
      allBodies.forEach((body) => {
        if (!body.isStatic && body.position.y > newH - 50) {
          Body.setPosition(body, { x: body.position.x, y: newH - 100 })
        }
      })
    }
  }

  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  Render.stop(render)
  Runner.stop(runner)
  Engine.clear(engine)
})
</script>

<template>
  <div class="fixed inset-0 bg-pink-50 overflow-hidden select-none touch-none">
    <div ref="canvas" class="w-full h-full cursor-grab active:cursor-grabbing border-none"></div>

    <Transition name="pop">
      <div
        v-if="selectedNote"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 backdrop-blur-sm p-4"
        @click="selectedNote = null"
      >
        <div class="bg-white rounded-3xl p-10 max-w-sm w-full shadow-2xl text-center" @click.stop>
          <p class="text-2xl text-gray-800 font-serif mb-8 italic">"{{ selectedNote }}"</p>
          <button
            @click="selectedNote = null"
            class="w-full bg-pink-500 text-white font-bold py-4 rounded-2xl shadow-lg hover:bg-pink-600 transition-colors"
          >
            Close
          </button>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.pop-enter-active,
.pop-leave-active {
  transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275);
}
.pop-enter-from,
.pop-leave-to {
  opacity: 0;
  transform: scale(0.8);
}
</style>
