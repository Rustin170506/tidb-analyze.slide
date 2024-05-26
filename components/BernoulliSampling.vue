<template>
  <div class="flex flex-col items-center space-y-4 text-white">
    <h1 class="text-2xl font-bold">Bernoulli Sampling</h1>
    <div class="flex space-x-2">
      <div
        v-for="(item, index) in dataset"
        :key="index"
        class="flex flex-col items-center"
      >
        <div
          :class="[
            'p-2 border rounded',
            sampledIndices.includes(index) ? 'bg-green-500' : 'bg-gray-800',
          ]"
        >
          {{ item }}
        </div>
        <div v-if="sampledIndices.includes(index)" class="mt-2 text-sm">
          Sampled
        </div>
      </div>
    </div>
    <div class="mt-4 text-xl">
      Sample Rate: {{ samplingProbability }}
    </div>
    <button @click="bernoulliSample" class="px-4 py-2 border rounded mt-4">
      Perform Bernoulli Sampling
    </button>
    <div v-if="sampledIndices.length > 0" class="mt-4 text-xl">
      Sampled Count: {{ sampledIndices.length }}
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue';

const dataset = [
  'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p',
];
const sampledIndices = ref([]);
const samplingProbability = ref(0.3);

const bernoulliSample = () => {
  sampledIndices.value = [];
  dataset.forEach((_, index) => {
    if (Math.random() < samplingProbability.value) {
      sampledIndices.value.push(index);
    }
  });
};
</script>
