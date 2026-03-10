<template>
    <div class="gallery__wrapper" ref="wrapperRef">
        <div class="gallery__track" ref="trackRef">
            <div 
            class="gallery__item" 
            v-for="(image, i) in images"
            :key="i">

            <img 
                :ref="el => { if (el) imageRefs[i] = el}"
                :src="image.src"
                :alt="image.alt"
                draggable="false"
                
            />

            </div>
        </div>

    </div>
</template>


<style scoped>
    .gallery__wrapper {
        width: 100vw;
        height: 100vh;
        overflow: hidden;
        display: flex;
        align-items: center;
    }

    .gallery__track {
        display: flex;
        gap: 1.5rem;
        padding: 0 5vw;
        will-change: transform;
    }

    .gallery__item {
        flex-shrink: 0;
        width: 38vw;
        height: 58vh;
        overflow: hidden;
        border-radius: 6px;
    }

    .gallery__item img {
        display: block;
        width: 125%;
        height: 100%;
        margin-left: -12.5%;
        object-fit: cover;
        will-change: transform;
        pointer-events: none;
        user-select: none;
    }

</style>

<script setup>


import { ref, onMounted, onUnmounted } from 'vue';
import gsap from 'gsap';

const wrapperRef = ref(null);
const trackRef = ref(null);
const imageRefs = ref[];

   const images = [
  { src: '/images/Serene Sand Dunes.png',              alt: 'Serene Sand Dunes' },
  { src: '/images/Serene Coastal Reflection.png',      alt: 'Serene Coastal Reflection' },
  { src: '/images/Dramatic Mountain Landscape.png',    alt: 'Dramatic Mountain Landscape' },
  { src: '/images/Misty Tropical Rainforest.png',      alt: 'Misty Tropical Rainforest' },
  { src: '/images/Abstract Cityscape Night.png',       alt: 'Abstract Cityscape Night' },
]


const clamp = gsap.utils.clamp;

const scroll = {
    current: 0,
    target: 0,
    ease: 0.07,
    limit: 0,

}

const MAX_SHIFT = 10


function setLimit() {
    scroll.limit = trackRef.value.scrollWidth - wrapperRef.value.clientWidth;
}


let tickerCb
let wheelCb
let resiszeCb

onMounted(() => {

    const setTrackX = gsap.quickSetter(trackRef.value, 'x', 'px');
    const setImageX = imageRefs.map(img => gsap.quickSetter(img, 'x', '%'));

    setLimit();

     tickerCb = () => {
    scroll.target  = clamp(0, scroll.limit, scroll.target)
    scroll.current += (scroll.target - scroll.current) * scroll.ease
    setTrackX(-scroll.current)
    applyParallax(setImageX)
  }

  wheelCb  = (e) => { scroll.target += e.deltaY }
  resizeCb = ()  => { setLimit() }

  gsap.ticker.add(tickerCb)
  gsap.ticker.lagSmoothing(0)
  window.addEventListener('wheel',  wheelCb,  { passive: true })
  window.addEventListener('resize', resizeCb, { passive: true })
})
</script>