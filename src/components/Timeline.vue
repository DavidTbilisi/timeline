<script setup>
import { events } from '../data/events.js'
</script>

<template>
  <div class="timeline">
    <div class="event" v-for="event in events" :key="event.id">
      <h3>{{ event.title }}</h3>
      <p>{{ event.start }} - {{ event.end }} BC</p>
      <p>{{ event.description }}</p>
    </div>
  </div>
</template>

<style scoped>
.timeline {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}
.event {
  border: 1px solid #ccc;
  padding: 1rem;
  border-radius: 4px;
}
</style>
