<template>
  <div>
    <div class="form-group text-center my-1">
      <div class="btn-group btn-group-xs">
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.TIME}" v-on:click="metric = metrics.TIME">time</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.ROWS}" v-on:click="metric = metrics.ROWS">rows</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.COST}" v-on:click="metric = metrics.COST">cost</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.SHARED}" v-on:click="metric = metrics.SHARED">{{ metrics.SHARED }}</button>
        <button class="btn btn-outline-secondary" :class="{'active': metric === metrics.TEMP}" v-on:click="metric = metrics.TEMP">{{ metrics.TEMP }}</button>
      </div>
    </div>
    <div class="legend text-center">
      <ul class="list-unstyled list-inline mb-0" v-if="metric == metrics.SHARED || metric == metrics.TEMP">
        <li class="list-inline-item">
          <span class="bg-hit rounded"></span>
          Hit
        </li>
        <li class="list-inline-item">
          <span class="bg-read"></span>
          Read
        </li>
        <li class="list-inline-item">
          <span class="bg-written"></span>
          Written
        </li>
        <li class="list-inline-item">
          <span class="bg-dirtied"></span>
          Dirtied
        </li>
      </ul>
    </div>
    <table class="my-1 table-hover">
      <tbody>
        <tr v-for="row, index in data" :content="tooltip(row[1])" v-tippy="{arrow: true, animation: 'fade', delay: [200, 0]}" @click.prevent="showNode(row[1], false, true)">
          <th class="node-type pr-2">
            <template v-for="i in lodash.range(row[0])">
              <template v-if="lodash.indexOf(row[3], i) != -1">│</template><template v-else>⠀</template>
            </template>{{ row[2] ? '└' : '├' }}
            {{ nodeType(row) }}
          </th>
          <td>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="metric == metrics.TIME">
              <div class="bg-secondary" role="progressbar" :style="'opacity: 0.2;width: ' + (row[1].node[nodeProps.INCLUSIVE_DURATION] - row[1].node[nodeProps.ACTUAL_DURATION]) / (plan.planStats.executionTime || plan.content.Plan[nodeProps.ACTUAL_TOTAL_TIME]) * 100 + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 1px;"></div>
              <div class="progress-bar border-left bg-secondary" role="progressbar" :style="'width: ' + row[1].node[nodeProps.ACTUAL_DURATION] / (plan.planStats.executionTime || plan.content.Plan[nodeProps.ACTUAL_TOTAL_TIME]) * 100 + '%; height:5px;'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="metric == metrics.ROWS">
              <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.ACTUAL_ROWS] / plan.planStats.maxRows * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="metric == metrics.COST">
              <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.ACTUAL_COST] / plan.planStats.maxCost * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="metric == metrics.SHARED && plan.planStats.maxSharedBlocks">
              <div class="bg-hit" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_HIT_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-read" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_READ_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-written" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_WRITTEN_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-dirtied" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.SHARED_DIRTIED_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
            </div>
            <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="metric == metrics.TEMP && plan.planStats.maxTempBlocks">
              <div class="bg-hit" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.TEMP_HIT_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-read" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.TEMP_READ_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-written" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.TEMP_WRITTEN_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              <div class="bg-dirtied" role="progressbar" :style="'width: ' + Math.round(row[1].node[nodeProps.TEMP_DIRTIED_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
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

  private metric: string = Metric.SHARED;

  private tooltip(cmp: PlanNode): string {
    switch (this.metric) {
      case Metric.TIME:
        return this.timeTooltip(cmp);
      case Metric.ROWS:
        return this.rowsTooltip(cmp);
      case Metric.COST:
        return this.costTooltip(cmp);
      case Metric.SHARED:
        return this.sharedBuffersTooltip(cmp);
      case Metric.TEMP:
        return this.tempBuffersTooltip(cmp);
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
    if (!this.plan.planStats.maxSharedBlocks) {
      return 'N/A';
    }
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

  private tempBuffersTooltip(cmp: PlanNode): string {
    if (!this.plan.planStats.maxTempBlocks) {
      return 'N/A';
    }
    let text = 'Temp Blocks:';
    const hit = cmp.node[NodeProp.TEMP_HIT_BLOCKS];
    text += hit !== 0 ? '<br>Hit: ' + rows(hit) : '';
    const read = cmp.node[NodeProp.TEMP_READ_BLOCKS];
    text += read !== 0 ? '<br>Read: ' + rows(read) : '';
    const written = cmp.node[NodeProp.TEMP_WRITTEN_BLOCKS];
    text += written !== 0 ? '<br>Written: ' + rows(written) : '';
    const dirtied = cmp.node[NodeProp.TEMP_DIRTIED_BLOCKS];
    text += dirtied !== 0 ? '<br>Dirtied: ' + rows(dirtied) : '';
    return text;
  }

  private nodeType(row: any[]): string {
    return row[1].node[NodeProp.NODE_TYPE];
  }
}
</script>
