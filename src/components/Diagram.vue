<template>
  <div>
    <div class="form-group text-center my-1">
      <div class="btn-group btn-group-xs">
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.TIME}" v-on:click="metric = metrics.TIME">time</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.ROWS}" v-on:click="metric = metrics.ROWS">rows</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.COST}" v-on:click="metric = metrics.COST">cost</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.SHARED_HIT}" v-on:click="metric = metrics.SHARED_HIT">buffers shared</button>
      </div>
    </div>
    <table class="my-1 table-hover">
      <tbody>
        <tr v-for="row in data" :content="tooltip(row[1])" v-tippy="{arrow: true, animation: 'fade', delay: [200, 0]}" @click.prevent="showNode(row[1], false, true)">
          <th class="node-type pr-2">
            <template v-for="i in lodash.range(row[0])">
              &nbsp;&nbsp;
            </template>
            {{ nodeType(row) }}
          </th>
          <td>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="metric == metrics.TIME">
              <div class="bg-secondary" role="progressbar" :style="'opacity: 0.2;width: ' + (row[1].node[nodeProps.INCLUSIVE_DURATION] - row[1].node[nodeProps.ACTUAL_DURATION]) / (plan.planStats.executionTime || plan.content.Plan[nodeProps.ACTUAL_TOTAL_TIME]) * 100 + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 1px;"></div>
              <div class="progress-bar border-left bg-secondary" role="progressbar" :style="'width: ' + row[1].node[nodeProps.ACTUAL_DURATION] / (plan.planStats.executionTime || plan.content.Plan[nodeProps.ACTUAL_TOTAL_TIME]) * 100 + '%; height:5px;'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="metric == metrics.ROWS">
              <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.ACTUAL_ROWS] / plan.planStats.maxRows * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="metric == metrics.COST">
              <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.ACTUAL_COST] / plan.planStats.maxCost * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="metric == metrics.SHARED_HIT">
              <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_HIT_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-success" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_READ_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-danger" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_WRITTEN_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-danger" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_DIRTIED_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import * as _ from 'lodash';
import { Component, Prop, Vue } from 'vue-property-decorator';
import { duration, durationClass, rows } from '@/filters';
import { NodeProp, Metric } from '../enums';
import PlanNode from '@/components/PlanNode.vue';
import { IPlan } from '../iplan';

@Component({
  filters: {
    durationClass,
  },
})
export default class Diagram extends Vue {
  @Prop() private data!: any[];
  @Prop() private plan!: IPlan;
  @Prop() private showNode!: () => void;

  private lodash = _;
  private nodeProps = NodeProp;
  private metrics = Metric;

  private metric: string = Metric.SHARED_HIT;

  private tooltip(cmp: PlanNode): string {
    switch (this.metric) {
      case Metric.TIME:
        return this.timeTooltip(cmp);
      case Metric.ROWS:
        return this.rowsTooltip(cmp);
      case Metric.COST:
        return this.costTooltip(cmp);
      case Metric.SHARED_HIT:
        return this.sharedBuffersTooltip(cmp);
    }
    return '';
  }

  private timeTooltip(cmp: PlanNode): string {
    return [
      'Duration: ',
      duration(cmp.node[NodeProp.ACTUAL_DURATION], true),
      ' | ',
      cmp.executionTimePercent,
      '%',
    ].join('');
  }

  private rowsTooltip(cmp: PlanNode): string {
    return [
      'Rows: ',
      rows(cmp.node[NodeProp.ACTUAL_ROWS]),
    ].join('');
  }

  private costTooltip(cmp: PlanNode): string {
    return [
      'Cost: ',
      rows(cmp.node[NodeProp.ACTUAL_COST]),
    ].join('');
  }

  private sharedBuffersTooltip(cmp: PlanNode): string {
    let text = 'Shared Blocks:';
    const hit = cmp.node[NodeProp.SHARED_HIT_BLOCKS];
    text += hit !== 0 ? '<br>Hit: ' + rows(hit) : '';
    const read = cmp.node[NodeProp.SHARED_READ_BLOCKS];
    text += read !== 0 ? '<br>Read: ' + rows(read) : '';
    const written = cmp.node[NodeProp.SHARED_WRITTEN_BLOCKS];
    text += written !== 0 ? '<br>Written: ' + rows(written) : '';
    const dirtied = cmp.node[NodeProp.SHARED_DIRTIED_BLOCKS];
    text += dirtied !== 0 ? '<br>Dirtied: ' + rows(dirtied) : '';
    return text;
  }

  private nodeType(row: any[]): string {
    return row[1].node[NodeProp.NODE_TYPE];
  }
}
</script>
