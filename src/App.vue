<script setup>
import { computed, ref } from 'vue'

const radius = ref(100)
const numpoints = ref(5)
const skips = ref(2)
const turnhardness = ref(10)
const turnangle = ref(0)
const seed = ref("seed")

const size = computed(() => (radius.value) * 3)
const center = computed(() => {return {x: size.value / 2, y: size.value / 2}})
const turnangleRad = computed(() => turnangle.value * Math.PI / 180)

const path = computed(computePath)

//onMounted(drawSvg)

function makePoints() {
  const points = []
  
  const d_angle = 2*Math.PI/numpoints.value

  let angle = 0
  for(let i = 0; i < numpoints.value; i++) {
    const point = {
      x: center.value.x + radius.value * Math.sin(angle),
      y: center.value.y + radius.value * Math.cos(angle)
    }
    points.push(point)
    angle += d_angle
  }
  return points
}

function computePath() {
  const sortedPoints = sortAndSkipPoints(makePoints(), skips.value, 0)
  return connectTheDots(sortedPoints)
}

function drawSvg() {
  const points = makePoints()
  
  console.log(points);

  const sortedPoints = sortAndSkipPoints(points, skips.value, 0)

  const ns = 'http://www.w3.org/2000/svg'
  const div = document.getElementById('drawing')
  div.innerHTML = ''
  const svg = document.createElementNS(ns, 'svg')
  svg.setAttributeNS(null, 'width', `${size.value}`)
  svg.setAttributeNS(null, 'height', `${size.value}`)
  div.appendChild(svg)
  const path = document.createElementNS(ns, 'path')
  path.setAttributeNS(null, 'd', connectTheDots(sortedPoints))
  path.setAttributeNS(null, 'fill', 'none')
  path.setAttributeNS(null, 'stroke', 'black')
  svg.appendChild(path)
  const projections = document.createElementNS(ns, 'path')
  projections.setAttributeNS(null, 'd', showProjections(sortedPoints))
  projections.setAttributeNS(null, 'fill', 'none')
  projections.setAttributeNS(null, 'stroke', 'red')
  //svg.appendChild(projections)

  console.log(sortedPoints);
}

function cyrb128() {
  const str = seed.value;
  let h1 = 1779033703, h2 = 3144134277,
      h3 = 1013904242, h4 = 2773480762;
  for (let i = 0, k; i < str.length; i++) {
      k = str.charCodeAt(i);
      h1 = h2 ^ Math.imul(h1 ^ k, 597399067);
      h2 = h3 ^ Math.imul(h2 ^ k, 2869860233);
      h3 = h4 ^ Math.imul(h3 ^ k, 951274213);
      h4 = h1 ^ Math.imul(h4 ^ k, 2716044179);
  }
  h1 = Math.imul(h3 ^ (h1 >>> 18), 597399067);
  h2 = Math.imul(h4 ^ (h2 >>> 22), 2869860233);
  h3 = Math.imul(h1 ^ (h3 >>> 17), 951274213);
  h4 = Math.imul(h2 ^ (h4 >>> 19), 2716044179);
  return (h1^h2^h3^h4)>>>0;
}

function mulberry32(a) {
    return function() {
      var t = a += 0x6D2B79F5;
      t = Math.imul(t ^ t >>> 15, t | 1);
      t ^= t + Math.imul(t ^ t >>> 7, t | 61);
      return ((t ^ t >>> 14) >>> 0) / 4294967296;
    }
}

function numToUTF16(num) {
    // we want to represent the input as a 8-bytes array
    var byteArray = [0, 0, 0, 0, 0, 0, 0, 0];

    for ( var index = 0; index < byteArray.length; index ++ ) {
        var byte = long & 0xffff;
        byteArray [ index ] = byte;
        long = (long - byte) / 256 ;
    }

    return byteArray;
};

function genSeed() {
  let seed = ''
  seed += String.fromCharCode(numpoints.value)
}

function randomize() {
  const rand = mulberry32(cyrb128())
  numpoints.value = Math.ceil(rand() * 20)
  skips.value = Math.ceil(rand() * Math.floor(numpoints.value / 2))
  turnangle.value = rand() * 361.0 - 180.0
  turnhardness.value = rand() * 100.0
}

function projectBezier(start, end, rev = false) {
  console.log(`${start} ${end}`)
  const rise = end.y - start.y
  const run = end.x - start.x
  let angle = Math.atan(rise/run)
  if(run < 0 || rev) {
    angle += Math.PI
  }

  const proj = {
    x: end.x + turnhardness.value * Math.cos(angle - turnangleRad.value),
    y: end.y + turnhardness.value * Math.sin(angle - turnangleRad.value)
  }
  return proj
}

function move(point) {
  return `M ${point.x} ${point.y}`
}

function line(point) {
  return `L ${point.x} ${point.y}`
}

function bezier_c(start, end, endend) {
  const proj_1 = projectBezier(endend, start, true)
  const proj_2 = projectBezier(start, end)
  return `C ${proj_1.x},${proj_1.y} ${proj_2.x},${proj_2.y} ${end.x},${end.y}`
}
function bezier(start, end) {
  const proj = projectBezier(start, end)
  return `S ${proj.x},${proj.y} ${end.x},${end.y}`
}

function connectTheDots(points) {
  let cmds = move(points[0]) + ' '
  cmds += bezier_c(points[0], points[1], points[points.length - 1])
  for(let i = 2; i < points.length; i++) {
    cmds += bezier(points[i - 1], points[i]) + ' '
  }
  cmds += bezier(points[points.length - 1], points[0]) + ' '
  return cmds
}
function showProjections(points) {
    const proj = projectBezier(points[points.length - 1], points[0])
    let cmds = move(points[0]) + ' ' + line(proj) + ' '
    for(let i = 1; i < points.length; i++) {
      const proj = projectBezier(points[i - 1], points[i])
      cmds += move(points[i]) + ' ' + line(proj) + ' '
    }
    return cmds
}

function sortAndSkipPoints(points, n, i) {
  // Create a new array to store the result
  const result = [];

  // Track the visited points to remove duplicates
  const visited = new Set();

  // Start with the given index and iterate over the points
  let currentIndex = i;

  while (result.length < points.length) {
    const currentPoint = points[currentIndex];

    // Add the current point to the result if it hasn't been visited
    if (visited.has(currentPoint)) {
      break
    }
    result.push(currentPoint);
    visited.add(currentPoint);

    // Move to the next point by skipping 'n' points
    currentIndex = (currentIndex + n) % points.length;
  }
  return result;
}



</script>
<style>
html {
  min-height: 100%;
  min-width: 100%;
  display: flex;
}
body {
  min-width: 100%;
}
label {
  display: inline-block;
  width: 100px;
  text-align: right;
}
/* .control {
  display: grid;
} */

.form {
  margin-top: 0;
}
span {
  width: 100px;
}
/* #drawing {
  width: 100%;
  height: 100%;
} */
.parent {
  margin-top: 0;
}
</style>
<template>
  <div class="parent">
    <div class="form">
      <!-- <div class="control">
        <label>radius</label>
        <input type=range v-model.number="radius" max = 100 >
        <span>{{radius}}</span>
      </div> -->
      <div class="control">
        <label>numpoints</label>
        <input type=range v-model.number="numpoints" max = 32 >
        <span>{{numpoints}}</span>
      </div>
      <div class="control">
        <label>skips</label>
        <input type=range v-model.number="skips" min=1 :max="numpoints/2" >
        <span>{{skips}}</span>
      </div>
      <div class="control">
        <label>turnangle</label>
        <input type=range v-model.number="turnangle" min="-180" max="180">
        <span>{{turnangle}}</span>
      </div>
      <div class="control">
        <label>turnhardness</label>
        <input type=range v-model.number="turnhardness">
        <span>{{turnhardness}}</span>
      </div>
      <input v-model="seed"/>
      <button @click="randomize">random</button>
    </div>
    <div id="drawing">
      <svg :width="size" :height="size"><path :d="path" fill="none" stroke="black"></path></svg>
    </div>
  </div>
</template>