<template>
  <div class="flex flex-col items-center space-y-4 text-black">
    <h1 class="text-2xl font-bold">Distinct Sampling</h1>
    <p>
      <span class="mb-1 mx-2"><strong>Sample Size:</strong> {{ 8 }}</span>
    </p>
    <div class="flex flex-wrap justify-center gap-2">
      <div v-for="(item, index) in dataset" :key="index" class="flex flex-col items-center">
        <div :class="['p-2 border rounded', currentIndex >= index ? 'bg-blue-200' : 'bg-gray-200']">{{ item }}</div>
        <div v-if="hashValues[index] !== null" class="mt-1 text-green-500">
          {{ '0'.repeat(countTrailingZeros(hashValues[index])) }}
        </div>
      </div>
    </div>
    <button @click="processNext" class="px-4 py-2 border rounded bg-blue-500 text-white">Process Next</button>
    <div class="mt-4 text-xl text-black">
      <p>Die Level: {{ mask }}</p>
      <p>Current Sample Size: {{ hashSet.size }}</p>
      <p>Estimated NDV: {{ estimatedNDV }}</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const MAX_SIZE = 8
const dataset = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'jj']
const hashValues = ref(Array(dataset.length).fill(null))
const mask = ref(0n)
const hashSet = ref(new Set())
const currentIndex = ref(-1)

const estimatedNDV = computed(() => {
  if (hashSet.value.size === 0) return 0
  return Number(mask.value + 1n) * hashSet.value.size
})

const hashFunction = (str) => {
  let h1 = 0xdeadbeefn
  let h2 = 0x41c6ce57n
  for (let i = 0; i < str.length; i++) {
    const ch = BigInt(str.charCodeAt(i))
    h1 = (h1 ^ ch) * 2654435761n
    h2 = (h2 ^ ch) * 1597334677n
  }
  h1 = ((h1 ^ (h1 >> 16n)) * 2246822507n) ^ ((h2 ^ (h2 >> 13n)) * 3266489909n)
  h2 = ((h2 ^ (h2 >> 16n)) * 2246822507n) ^ ((h1 ^ (h1 >> 13n)) * 3266489909n)
  return (h2 << 32n) | h1
}

const countTrailingZeros = (n) => {
  if (n === 0n) return 64 // For 64-bit numbers
  let count = 0
  while ((n & 1n) === 0n) {
    n >>= 1n
    count++
  }
  return count
}

const processNext = () => {
  if (currentIndex.value < dataset.length - 1) {
    currentIndex.value++
    const item = dataset[currentIndex.value]
    const hash = hashFunction(item)
    hashValues.value[currentIndex.value] = hash

    if ((hash & mask.value) === 0n) {
      hashSet.value.add(hash)
      if (hashSet.value.size > MAX_SIZE) {
        const newMask = (mask.value << 1n) | 1n
        hashSet.value = new Set([...hashSet.value].filter(x => (x & newMask) === 0n))
        mask.value = newMask
      }
    }
  }
}
</script>
