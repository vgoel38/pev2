<template>
  <i class="fa fa-exclamation-triangle" :class="cls" v-if="cls"></i>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

@Component
export default class HighlightIcon extends Vue {
  @Prop() public cls!: string | boolean;
}
</script>
