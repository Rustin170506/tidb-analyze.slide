<template>
  <div class="flex flex-col items-center w-full max-w-4xl mx-auto">
    <h1 class="text-2xl font-bold">Build TopN</h1>
    <div class="flex justify-between w-full mb-8">
      <div class="w-1/2 pr-4">
        <h3 class="text-lg font-bold mb-2">Samples</h3>
        <div class="flex flex-wrap">
          <div v-for="(sample, index) in sortedSamples" :key="index" class="p-2 border rounded m-1"
            :class="{ 'bg-yellow-200': currentIndex === index }">
            {{ sample }}
          </div>
        </div>
      </div>
      <div class="w-1/2 pl-4">
        <h3 class="text-lg font-bold mb-2">TopN List</h3>
        <div class="flex flex-wrap">
          <div v-for="(item, index) in topNList" :key="index" class="p-2 border rounded m-1 flex items-center">
            <span class="mr-2">{{ item.value }}:</span>
            <span>{{ item.count }}</span>
          </div>
        </div>
      </div>
    </div>

    <div class="w-full text-left mb-4">
      <p><strong>Current:</strong> {{ current }}</p>
      <p><strong>Current Count:</strong> {{ currentCount }}</p>
      <p><strong>Step:</strong> {{ stepDescription }}</p>
    </div>

    <div class="flex space-x-4">
      <button @click="step" class="px-4 py-2 bg-blue-500 text-white rounded">Next Step</button>
      <button @click="reset" class="px-4 py-2 bg-gray-500 text-white rounded">Reset</button>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'

const originalSamples = ['B', 'A', 'C', 'B', 'D', 'A', 'B', 'E', 'D']
const numTopN = 3

const currentIndex = ref(-1)
const topNList = ref([])
const current = ref('')
const currentCount = ref(0)
const stepDescription = ref('')
const isSorted = ref(false)

const sortedSamples = computed(() => {
  if (isSorted.value) {
    return [...originalSamples].sort()
  }
  return originalSamples
})

const step = () => {
  if (!isSorted.value) {
    isSorted.value = true
    stepDescription.value = 'Samples sorted'
    return
  }

  if (currentIndex.value < sortedSamples.value.length - 1) {
    currentIndex.value++
    const sample = sortedSamples.value[currentIndex.value]

    if (sample === current.value) {
      currentCount.value++
      stepDescription.value = `Incrementing count for ${sample}`
    } else {
      if (topNList.value.length === 0 || currentCount.value > topNList.value[topNList.value.length - 1].count) {
        let insertIndex = topNList.value.findIndex(item => currentCount.value > item.count)
        if (insertIndex === -1) insertIndex = topNList.value.length

        topNList.value.splice(insertIndex, 0, { value: current.value, count: currentCount.value })
        if (topNList.value.length > numTopN) {
          topNList.value.pop()
        }
        stepDescription.value = `Inserted ${current.value} into TopN list`
      } else {
        stepDescription.value = `${current.value} not frequent enough for TopN list`
      }
      current.value = sample
      currentCount.value = 1
    }
  } else {
    // Handle the last element
    if (currentCount.value > 0) {
      let insertIndex = topNList.value.findIndex(item => currentCount.value > item.count)
      if (insertIndex === -1) insertIndex = topNList.value.length

      topNList.value.splice(insertIndex, 0, { value: current.value, count: currentCount.value })
      if (topNList.value.length > numTopN) {
        topNList.value.pop()
      }
      stepDescription.value = `Inserted last element ${current.value} into TopN list`
    }
    current.value = ''
    currentCount.value = 0
  }

  if (currentIndex.value === sortedSamples.value.length - 1 && current.value === '') {
    stepDescription.value = "Finished"
  }
}

const reset = () => {
  currentIndex.value = -1
  topNList.value = []
  current.value = ''
  currentCount.value = 0
  stepDescription.value = ''
  isSorted.value = false
}

</script>
