<template>
    <v-card
        class="mx-2"
        :color="msg.role === 'system' ? 'primary' : 'grey-lighten-3'"
        elevation="4"
        max-width="45%"
    >
        
        <v-card-text>
            <v-skeleton-loader
             v-if="msg.isLoading"
             type="list-item-two-line"
             color="primary"
             min-width="500px">
            </v-skeleton-loader>
            <span v-else>{{ msg.content}}</span>
        </v-card-text>
    </v-card>
</template>

<script setup lang="ts">
    import { IMessage } from "../models/message";
    defineProps<{
        msg: IMessage
    }>()

</script>

<style scoped>

</style>