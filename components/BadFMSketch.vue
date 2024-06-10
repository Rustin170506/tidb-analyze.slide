<template>
  <div class="flex flex-col items-center space-y-4 text-black">
    <h1 class="text-2xl font-bold">Flajolet-Martin Sketch - A Bad Case</h1>
    <div class="flex space-x-2">
      <div v-for="(item, index) in dataset" :key="index" class="flex flex-col items-center">
        <div class="p-2 border rounded bg-gray-200">{{ item }}</div>
        <div class="mt-2 text-sm">{{ hashValues[index] }}</div>
        <div v-if="hashValues[index]" class="mt-1">
          <div v-for="n in tailZeros[index]" :key="n" class="text-green-500">0</div>
        </div>
      </div>
    </div>
    <button @click="generateHashValues" class="px-4 py-2 border rounded bg-blue-500 text-white">Generate Hash Values</button>
    <div v-if="maxTailZeros !== null" class="mt-4 text-xl text-black">
      Estimated Cardinality: 2^{{ maxTailZeros }} = {{ 2 ** maxTailZeros }}
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

const dataset = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'jj']
const hashValues = ref(Array(dataset.length).fill(null))
const tailZeros = ref(Array(dataset.length).fill(0))
const maxTailZeros = ref(null)

const hashFunction = (str) => {
  let hash = 0
  for (let i = 0; i < str.length; i++) {
    const char = str.charCodeAt(i)
    hash = (hash << 5) - hash + char
    hash |= 0 // Convert to 32bit integer
  }
  return hash >>> 0 // Convert to unsigned 32bit integer
}

const countTrailingZeros = (num) => {
  let count = 0
  while ((num & 1) === 0) {
    count++
    num >>= 1
  }
  return count
}

const generateHashValues = () => {
  let maxZeros = 0
  dataset.forEach((item, index) => {
    const hashValue = hashFunction(item)
    hashValues.value[index] = hashValue
    const zeros = countTrailingZeros(hashValue)
    tailZeros.value[index] = zeros
    if (zeros > maxZeros) {
      maxZeros = zeros
    }
  })
  maxTailZeros.value = maxZeros
}
</script>
