<script setup>
import { ref } from "vue";
import { log } from "@/services/utility.js"; // logging browser service
import HelloWorld from '@/components/HelloWorld.vue'
import CreateStream from '@/components/CreateStream.vue'
import VideoStream from '@/components/VideoStream.vue'

let isShow = ref(null)

function changeShow(show) {
    isShow.value = show
}

</script>

<template>
    <div>
        <header>
            <img alt="App logo" class="logo" src="@/assets/images/logo.png" width="60" height="60" />
        </header>
        <div v-if="isShow == null">
            <button class="button" @click="changeShow(false)">Почати стрім</button>
            <button class="button" @click="changeShow(true)">Переглянути стрім</button>
        </div>
        <CreateStream v-if="isShow != null && !isShow" />
        <VideoStream v-if="isShow" />
    </div>
<!--     <div>
        <video ref="videoElement" width="640" height="360" autoplay></video>
  </div> -->
  
</template>

<style>
@import './assets/css/base.css';

#app {
    max-width: 1680px;
    margin: 0 auto;
    padding: 2rem;

    font-weight: normal;
}

header {
    line-height: 1.5;
}

.list {
    padding: 0 15px;
}

.logo {
    display: block;
    margin: 0 auto 2rem;
}

a,
.blue {
    text-decoration: none;
    color: #0070ff;
    transition: 0.4s;
}

@media (hover: hover) {
    a:hover {
        background-color: hsla(160, 100%, 37%, 0.2);
    }
}

@media (min-width: 1024px) {
    body {
        display: flex;
        place-items: center;
    }

    #app {
        display: grid;
        grid-template-columns: 1fr 1fr;
        padding: 0 2rem;
    }

    header {
        display: flex;
        place-items: center;
        padding-right: calc(var(--section-gap) / 2);
    }

    header .wrapper {
        display: flex;
        place-items: flex-start;
        flex-wrap: wrap;
    }

    .logo {
        margin: 0 2rem 0 0;
    }
}
</style>