<template>
  <div class="flex flex-col">
    <PageTitle title="Knowledge base" />
    <div class="flex grow border-t">
      <KnowledgeBaseSidebar />
      <RouterView :key="route.fullPath" v-slot="{ Component }">
        <component :is="Component" v-if="Component" />
        <KnowledgeBaseEmptyMessage v-else message="Select a category" />
      </RouterView>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useRoute } from "vue-router";
import PageTitle from "@/components/PageTitle.vue";
import KnowledgeBaseEmptyMessage from "./KnowledgeBaseEmptyMessage.vue";
import KnowledgeBaseSidebar from "./KnowledgeBaseSidebar.vue";

const route = useRoute();
</script>
