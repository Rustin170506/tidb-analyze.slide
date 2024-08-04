<template>
    <div class="flex flex-col items-center w-full max-w-4xl mx-auto">
        <h1 class="text-2xl font-bold">Build Histogram</h1>
        <p>
            <span class="mb-1 mx-2"><strong>Count:</strong> {{ count }}</span>
            <span class="mb-1 mx-2"><strong>Sample Length:</strong> {{ samples.length }}</span>
        </p>
        <div class="chart-container w-full mb-8" style="height: 250px;">
            <Bar :data="chartData" :options="chartOptions" />
        </div>

        <div class="w-full text-center flex flex-col items-center">
            <p>
                <span class="mb-1 mx-2"><strong>Sample Factor:</strong> {{ sampleFactor }}</span>
                <span class="mb-1 mx-2"><strong>Per Bucket:</strong> {{ valuesPerBucket.toFixed(1) }}</span>
                <span class="mb-1 mx-2"><strong>Current Sample:</strong> {{ currentSample ? currentSample :
                    'N/A' }}</span>
                <span class="mb-1 mx-2"><strong>Current Bucket:</strong> {{ currentBucket + 1 }}</span>
            </p>
            <p><strong>Step:</strong> {{ stepDescription ? stepDescription : 'Click Next Step to start' }}</p>
        </div>

        <div class="flex space-x-4">
            <button @click="step" class="px-4 py-2 bg-blue-500 text-white rounded">Next Step</button>
            <button @click="reset" class="px-4 py-2 bg-gray-500 text-white rounded">Reset</button>
        </div>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { Bar } from 'vue-chartjs'
import {
    Chart as ChartJS,
    Title,
    Tooltip,
    Legend,
    BarElement,
    CategoryScale,
    LinearScale
} from 'chart.js'

ChartJS.register(CategoryScale, LinearScale, BarElement, Title, Tooltip, Legend)

const samples = [5, 13, 23, 34, 45, 57, 69, 80, 92, 104, 116, 128, 140, 152, 164, 176, 188, 200, 212, 224]
const numBuckets = 3
const count = 200
const ndv = 100

const currentIndex = ref(-1)
const currentSample = ref<number | null>(null)
const currentBucket = ref(0)
const stepDescription = ref('')
const histogram = ref<Array<{ lower_bound: number, upper_bound: number, count: number, repeats: number }>>([])

const sampleFactor = count / samples.length
const ndvFactor = Math.min(count / ndv, sampleFactor)
const valuesPerBucket = count / numBuckets + sampleFactor

const initializeHistogram = () => {
    histogram.value = Array(numBuckets).fill(null).map(() => ({
        lower_bound: 0,
        upper_bound: 0,
        count: 0,
        repeats: 0
    }))
}

const chartData = computed(() => ({
    labels: histogram.value.map((_, index) => `Bucket ${index + 1}`),
    datasets: [{
        data: histogram.value.map(bucket => bucket.count),
        borderColor: "gray",
        borderWidth: 1,
    }]
}))

const step = () => {
    if (currentIndex.value < samples.length - 1) {
        currentIndex.value++
        currentSample.value = samples[currentIndex.value]

        if (currentIndex.value === 0) {
            histogram.value[0] = {
                lower_bound: currentSample.value,
                upper_bound: currentSample.value,
                count: sampleFactor,
                repeats: ndvFactor
            }
            stepDescription.value = `Appended first sample ${currentSample.value} to histogram`
        } else {
            const bucket = histogram.value[currentBucket.value]
            const upper = bucket.upper_bound
            const cmp = currentSample.value - upper

            if (cmp === 0) {
                bucket.count += sampleFactor
                bucket.repeats += sampleFactor
                stepDescription.value = `Updated count for bucket ${currentBucket.value + 1}`
            } else if (bucket.count <= valuesPerBucket) {
                bucket.upper_bound = currentSample.value
                bucket.count += sampleFactor
                bucket.repeats = ndvFactor
                stepDescription.value = `Updated upper bound and count for bucket ${currentBucket.value + 1}`
            } else {
                currentBucket.value++
                histogram.value[currentBucket.value] = {
                    lower_bound: currentSample.value,
                    upper_bound: currentSample.value,
                    count: 0,
                    repeats: ndvFactor
                }
                stepDescription.value = `Created new bucket ${currentBucket.value + 1}`
            }
        }
    } else {
        stepDescription.value = 'Histogram construction complete'
    }
}

const reset = () => {
    currentIndex.value = -1
    currentSample.value = null
    currentBucket.value = 0
    stepDescription.value = ''
    initializeHistogram()
}

const chartOptions = computed(() => {
    return {
        responsive: true,
        maintainAspectRatio: false,
        scales: {
            y: {
                beginAtZero: true,
                ticks: {
                    color: "darkblue",
                },
            },
            x: {
                ticks: {
                    color: "darkblue",
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
                        const item = histogram.value[context.dataIndex];
                        return `ID: ${context.dataIndex + 1}, Count: ${item.count}`;
                    },
                    afterBody: function (context) {
                        const index = context[0].dataIndex;
                        const item = histogram.value[index];
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
                color: "darkblue",
            },
            legend: {
                display: false,
            },
        },
    }
})

onMounted(() => {
    initializeHistogram()
})
</script>

<style scoped>
.chart-container {
    max-width: 800px;
    margin: 0 auto;
}
</style>
