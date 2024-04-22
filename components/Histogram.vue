<template>
  <div>
    <canvas ref="canvasRef"></canvas>
  </div>
</template>

<script setup lang="ts">
import { Chart, registerables } from "chart.js";
import { onMounted, ref } from "vue";
// @ts-ignore
import histogramData from "./histogram.json";

Chart.register(...registerables);

const canvasRef = ref<HTMLCanvasElement | null>(null);

onMounted(() => {
  const canvas = canvasRef.value;
  if (!canvas) {
    console.error("Canvas element not found");
    return;
  }

  const ctx = canvas.getContext("2d");
  if (!ctx) {
    console.error("Failed to get 2D context for canvas");
    return;
  }

  new Chart(ctx, {
    type: "bar",
    data: {
      labels: histogramData.map((item) => item.lower_bound),
      datasets: [
        {
          data: histogramData.map((item) => item.count),
          borderColor: "white",
          borderWidth: 1,
        },
      ],
    },
    options: {
      scales: {
        y: {
          beginAtZero: true,
          ticks: {
            color: "white",
          },
        },
        x: {
          ticks: {
            color: "white",
          },
        },
      },
      plugins: {
        tooltip: {
          callbacks: {
            title: function () {
              return "Histogram Bucket";
            },
            label: function (context) {
              const item = histogramData[context.dataIndex];
              return `ID: ${item.bucket_id}, Count: ${item.count}`;
            },
            afterBody: function (context) {
              const index = context[0].dataIndex;
              const item = histogramData[index];
              return [
                `LowerBound: ${item.lower_bound}`,
                `UpperBound: ${item.upper_bound}`,
                `Repeats: ${item.repeats}`,
              ];
            },
          },
        },
        title: {
          display: true,
          text: "Histogram",
          color: "white",
        },
        legend: {
          display: false,
        },
      },
    },
  });
});
</script>
