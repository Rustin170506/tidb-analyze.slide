<template>
    <div class="relative w-26 h-4 bg-gray-200 overflow-hidden">
        <canvas ref="canvas" class="absolute inset-0 cursor-pointer"></canvas>
        <div ref="textDiv" class="absolute inset-0 flex items-center justify-center text-xs font-bold invisible">
            🤡
        </div>
    </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

const canvas = ref(null);
const textDiv = ref(null);

onMounted(() => {
    const container = canvas.value.parentElement;
    const ctx = canvas.value.getContext('2d');

    canvas.value.width = container.offsetWidth;
    canvas.value.height = container.offsetHeight;

    ctx.fillStyle = '#888';
    ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);

    let isDrawing = false;

    const draw = (e) => {
        if (!isDrawing) return;
        const rect = canvas.value.getBoundingClientRect();
        const x = e.clientX - rect.left;
        const y = e.clientY - rect.top;
        ctx.globalCompositeOperation = 'destination-out';
        ctx.beginPath();
        ctx.arc(x, y, 5, 0, 2 * Math.PI);
        ctx.fill();

        checkReveal();
    };

    const checkReveal = () => {
        const imageData = ctx.getImageData(0, 0, canvas.value.width, canvas.value.height);
        const pixels = imageData.data;
        let transparent = 0;
        for (let i = 0; i < pixels.length; i += 4) {
            if (pixels[i + 3] < 128) transparent++;
        }
        if (transparent > pixels.length / 4 * 0.5) {
            textDiv.value.classList.remove('invisible');
        }
    };

    canvas.value.addEventListener('mousedown', (e) => { isDrawing = true; draw(e); });
    canvas.value.addEventListener('mousemove', draw);
    canvas.value.addEventListener('mouseup', () => isDrawing = false);
    canvas.value.addEventListener('mouseout', () => isDrawing = false);
});
</script>
