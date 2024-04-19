<template>
  <div>
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<script setup lang="ts">
import { Chart, registerables } from 'chart.js';
Chart.register(...registerables);

import { onMounted, ref } from 'vue';

const canvasRef = ref<HTMLCanvasElement | null>(null);

onMounted(() => {
  const canvas = canvasRef.value;
  if (!canvas) {
    console.error('Canvas element not found');
    return;
  }

  const ctx = canvas.getContext('2d');
  if (!ctx) {
    console.error('Failed to get 2D context for canvas');
    return;
  }

  const data = Array.from({ length: 1000 }, () => Math.random() * 100);
  const numBins = 20;

  new Chart(ctx, {
    type: 'bar',
    data: {
      labels: Array.from({ length: numBins }, (_, i) => `Bin ${i + 1}`),
      datasets: [{
        data,
        backgroundColor: 'rgba(75, 192, 192, 0.2)',
        borderColor: 'rgba(75, 192, 192, 1)',
        borderWidth: 1
      }]
    },
    options: {
      scales: {
        y: {
          beginAtZero: true
        }
      }
    }
  });
});
</script>
