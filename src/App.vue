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
  'Ikaw ang pinakamagandang babae na nakilala! 😘',
  'Nami-miss ko na ang imong kiss, Love! 🥲',
  'Sorry for everything. 🥺',
  'Hmmmmm.... Love mo ba ako? 🤔',
  'Sapakan nga tayo babe, one time lang. 🤣',
  "I'm so proud of you! 😘",
  'Mahal na mahal kita sobra, kaya huwag ka nang magselos dahil ikaw lang ang mahal ko palagi. 🤩',
  'Wanna hug you so bad, Love 🤗',
  'I wanna feel your lips against mine 😚',
  'Your eyes are so beautiful. One of a kind. Sana sa akin lang nakatingin 😏',
  "I'm always here for you. Sasamahan kita sa anumang hamon sa buhay.",
  "You're so pretty! 😍",
  'Sana tayo na habang buhay! ☺️',
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
    return Bodies.rectangle(w / 2 + i * 10, h - 100 - i * 20, 140, 40, {
      label: text,
      chamfer: { radius: 12 },
      render: {
        fillStyle: ['#fb7185', '#f472b6', '#f9a8d4', '#fda4af'][i % 4],
        strokeStyle: '#ffffff',
        lineWidth: 3,
      },
    })
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

  const handleResize = () => {
    const newW = window.innerWidth
    const newH = window.innerHeight

    render.canvas.width = newW
    render.canvas.height = newH
    render.options.width = newW
    render.options.height = newH

    if (floor && ceiling && wallLeft && wallRight) {
      Body.setPosition(floor, { x: newW / 2, y: newH + thickness / 2 })
      Body.setPosition(ceiling, { x: newW / 2, y: -thickness / 2 })
      Body.setPosition(wallLeft, { x: -thickness / 2, y: newH / 2 })
      Body.setPosition(wallRight, { x: newW + thickness / 2, y: newH / 2 })

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
  <div
    class="fixed inset-0 bg-pink-50 overflow-hidden select-none touch-none flex flex-col items-center"
  >
    <div class="absolute top-10 px-6 text-center z-20 pointer-events-none">
      <h1 class="text-3xl md:text-4xl font-serif text-pink-600 font-bold mb-2 drop-shadow-sm">
        A Jar of Love for You 🏺
      </h1>
      <p class="text-pink-400 text-sm md:text-base italic max-w-md animate-float">
        Whenever you feel that our love is fading, or life gets too heavy... just reach in and read
        one. I'm always here for you.
      </p>
    </div>

    <div ref="canvas" class="w-full h-full cursor-grab active:cursor-grabbing border-none"></div>

    <Transition name="pop">
      <div
        v-if="selectedNote"
        class="fixed inset-0 z-50 flex items-center justify-center bg-black/40 backdrop-blur-sm p-4"
        @click="selectedNote = null"
      >
        <div
          class="bg-white rounded-[2.5rem] p-10 max-w-sm w-full shadow-2xl text-center border-b-8 border-pink-200"
          @click.stop
        >
          <div class="text-5xl mb-6">💌</div>
          <p class="text-2xl text-gray-800 font-serif mb-10 italic leading-relaxed">
            "{{ selectedNote }}"
          </p>
          <button
            @click="selectedNote = null"
            class="w-full bg-pink-500 text-white font-bold py-4 rounded-2xl shadow-lg hover:bg-pink-600 transition-all active:scale-95"
          >
            I love you too
          </button>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.animate-float {
  animation: float 4s ease-in-out infinite;
}

@keyframes float {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-10px);
  }
}

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
