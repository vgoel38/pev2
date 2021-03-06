<template>
  <div>
    <div class="form-group text-center my-1">
      <div class="btn-group btn-group-xs">
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.metric === metrics.time}" v-on:click="viewOptions.metric = metrics.time">time</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.metric === metrics.rows}" v-on:click="viewOptions.metric = metrics.rows">rows</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.metric === metrics.estimate_factor}" v-on:click="viewOptions.metric = metrics.estimate_factor">estimation</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.metric === metrics.cost}" v-on:click="viewOptions.metric = metrics.cost">cost</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.metric === metrics.buffers}" v-on:click="viewOptions.metric = metrics.buffers">buffers</button>
      </div>
    </div>
    <div class="form-group text-center my-1" v-if="viewOptions.metric == metrics.buffers">
      <div class="btn-group btn-group-xs">
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.buffersMetric === buffersMetrics.shared}" v-on:click="viewOptions.buffersMetric = buffersMetrics.shared">shared</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.buffersMetric === buffersMetrics.temp}" v-on:click="viewOptions.buffersMetric = buffersMetrics.temp">temp</button>
        <button class="btn btn-outline-secondary" :class="{'active': viewOptions.buffersMetric === buffersMetrics.local}" v-on:click="viewOptions.buffersMetric = buffersMetrics.local">local</button>
      </div>
    </div>
    <div class="legend text-center">
      <ul class="list-unstyled list-inline mb-0" v-if="viewOptions.metric == metrics.buffers">
        <li class="list-inline-item">
          <span class="bg-hit rounded"></span>
          Hit
        </li>
        <li class="list-inline-item">
          <span class="bg-read"></span>
          Read
        </li>
        <li class="list-inline-item">
          <span class="bg-dirtied"></span>
          Dirtied
        </li>
        <li class="list-inline-item">
          <span class="bg-written"></span>
          Written
        </li>
      </ul>
    </div>
    <table class="my-1 table-hover">
      <tbody v-for="flat, index in plans">
        <tr v-if="index === 0 && plans.length > 1">
          <th colspan="2" class="subplan">
            Main Query Plan
          </th>
        </tr>
        <template v-for="row, index in flat">
          <tr v-if="row[1][nodeProps.SUBPLAN_NAME]">
            <td class="subplan pr-2" :class="{'font-weight-bold': lodash.startsWith(row[1][nodeProps.SUBPLAN_NAME], 'CTE')}" colspan="2" @click.prevent="showCTE(row[1][nodeProps.SUBPLAN_NAME])">
              <template v-for="i in lodash.range(row[0])">
                <template v-if="lodash.indexOf(row[3], i) != -1">│</template><template v-else-if="i !== 0">⠀</template>
              </template>
              <template v-if="index !== 0">{{ row[2] ? '└' : '├' }}</template>
              {{ row[1][nodeProps.SUBPLAN_NAME] }}
            </td>
          </tr>
          <tr :content="tooltip(row[1])" v-tippy="{arrow: true, animation: 'fade', delay: [200, 0]}" @click.prevent="showNode(row[1], false, true)">
            <td class="node-type pr-2">
              <template v-for="i in lodash.range(row[0])">
                <template v-if="lodash.indexOf(row[3], i) != -1">│</template><template v-else-if="i !== 0">⠀</template>
              </template>
              <template v-if="index !== 0">
                <template v-if="!row[1][nodeProps.SUBPLAN_NAME]">{{ row[2] ? '└' : '├' }}</template>
                <template v-else>
                  <template v-if="!row[2]">│</template><template v-else>⠀</template>
                </template>
              </template>
              {{ nodeType(row) }}
            </td>
            <td>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-if="viewOptions.metric == metrics.time">
                <div class="progress-bar border-left bg-secondary" role="progressbar" :style="'width: ' + row[1][nodeProps.ACTUAL_DURATION] / (plan.planStats.executionTime || plan.content.Plan[nodeProps.ACTUAL_TOTAL_TIME]) * 100 + '%; height:5px;'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="viewOptions.metric == metrics.rows">
                <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.ACTUAL_ROWS] / plan.planStats.maxRows * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent justify-content-center" style="height: 10px;" v-if="viewOptions.metric == metrics.estimate_factor">
                <span class="text-muted small">
                  <i class="fa fa-fw" :class="{'fa-arrow-down': row[1][nodeProps.PLANNER_ESTIMATE_DIRECTION] === estimateDirections.under}"></i>
                </span>
                <div class="progress-bar" :class="[row[1][nodeProps.PLANNER_ESTIMATE_DIRECTION] === estimateDirections.under ? 'bg-secondary' : 'bg-transparent']" role="progressbar" :style="'width: ' + (row[1][nodeProps.PLANNER_ESTIMATE_FACTOR] || 0) / maxEstimateFactor * 100 + '%; height:5px;'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
                <div class="progress-bar border-left" role="progressbar" style="width: 1px; height: 5px;" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
                <div class="progress-bar" :class="[row[1][nodeProps.PLANNER_ESTIMATE_DIRECTION] === estimateDirections.over ? 'bg-secondary' : 'bg-transparent']" role="progressbar" :style="'width: ' + (row[1][nodeProps.PLANNER_ESTIMATE_FACTOR] || 0) / maxEstimateFactor * 100 + '%; height:5px;'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100"></div>
                <span class="text-muted small">
                  <i class="fa fa-fw" :class="{'fa-arrow-up': row[1][nodeProps.PLANNER_ESTIMATE_DIRECTION] === estimateDirections.over}"></i>
                </span>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="viewOptions.metric == metrics.cost">
                <div class="bg-secondary" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.ACTUAL_COST] / plan.planStats.maxCost * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="viewOptions.metric == metrics.buffers && viewOptions.buffersMetric == buffersMetrics.shared && plan.planStats.maxSharedBlocks">
                <div class="bg-hit" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.SHARED_HIT_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-read" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.SHARED_READ_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-dirtied" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.SHARED_DIRTIED_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-written" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.SHARED_WRITTEN_BLOCKS] / plan.planStats.maxSharedBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="viewOptions.metric == metrics.buffers && viewOptions.buffersMetric == buffersMetrics.temp && plan.planStats.maxTempBlocks">
                <div class="bg-hit" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.TEMP_HIT_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-read" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.TEMP_READ_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-dirtied" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.TEMP_DIRTIED_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-written" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.TEMP_WRITTEN_BLOCKS] / plan.planStats.maxTempBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              </div>
              <div class="progress rounded-0 align-items-center bg-transparent" style="height: 5px;" v-else-if="viewOptions.metric == metrics.buffers && viewOptions.buffersMetric == buffersMetrics.local && plan.planStats.maxLocalBlocks">
                <div class="bg-hit" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.LOCAL_HIT_BLOCKS] / plan.planStats.maxLocalBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-read" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.LOCAL_READ_BLOCKS] / plan.planStats.maxLocalBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-dirtied" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.LOCAL_DIRTIED_BLOCKS] / plan.planStats.maxLocalBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
                <div class="bg-written" role="progressbar" :style="'width: ' + Math.round(row[1][nodeProps.LOCAL_WRITTEN_BLOCKS] / plan.planStats.maxLocalBlocks * 100) + '%'" aria-valuenow="15" aria-valuemin="0" aria-valuemax="100" style="height: 5px;"></div>
              </div>
            </td>
          </tr>
        </template>
      </tbody>
    </table>
  </div>
</template>

<script lang="ts">
import * as _ from 'lodash';
import { Component, Prop, Vue, Watch } from 'vue-property-decorator';
import { duration, durationClass, rows, factor } from '@/filters';
import { EstimateDirection, BuffersMetric, NodeProp, Metric } from '../enums';
import Node from '@/inode';
import { IPlan } from '../iplan';

@Component({
  filters: {
    durationClass,
  },
})
export default class Diagram extends Vue {
  @Prop() private plan!: IPlan;
  @Prop() private showNode!: () => void;
  @Prop() private showCTE!: () => void;

  // The main plan + init plans (all flatten)
  private plans: any[][] = [[]];
  private lodash = _;
  private nodeProps = NodeProp;
  private metrics = Metric;
  private buffersMetrics = BuffersMetric;
  private estimateDirections = EstimateDirection;

  private viewOptions: any = {
    metric: Metric.time,
    buffersMetric: BuffersMetric.shared,
  };

  private created(): void {
    const savedOptions = localStorage.getItem('diagramViewOptions');
    if (savedOptions) {
      _.assignIn(this.viewOptions, JSON.parse(savedOptions));
    }
    this.flatten(this.plans[0], 0, this.plan.content.Plan, true, []);

    _.each(this.plan.ctes, (cte) => {
      const flat = [] as any[];
      this.flatten(flat, 0, cte, true, []);
      this.plans.push(flat);
    });
  }

  @Watch('viewOptions', {deep: true})
  private onViewOptionsChanged(val: any, oldVal: any) {
    localStorage.setItem('diagramViewOptions', JSON.stringify(this.viewOptions));
  }

  private tooltip(node: Node): string {
    let content = '';
    switch (this.viewOptions.metric) {
      case Metric.time:
        content += this.timeTooltip(node);
        break;
      case Metric.rows:
        content += this.rowsTooltip(node);
        break;
      case Metric.estimate_factor:
        content += this.estimateFactorTooltip(node);
        break;
      case Metric.cost:
        content += this.costTooltip(node);
        break;
      case Metric.buffers:
        content += this.buffersTooltip(node);
        break;
    }
    if (node[NodeProp.CTE_NAME]) {
      content +=  '<br><em>CTE ' + node[NodeProp.CTE_NAME] + '</em>';
    }
    return content;
  }

  private timeTooltip(node: Node): string {
    return [
      'Duration: ',
      duration(node[NodeProp.ACTUAL_DURATION], true),
    ].join('');
  }

  private rowsTooltip(node: Node): string {
    return [
      'Rows: ',
      rows(node[NodeProp.ACTUAL_ROWS]),
    ].join('');
  }

  private estimateFactorTooltip(node: Node): string {
    const estimateFactor = node[NodeProp.PLANNER_ESTIMATE_FACTOR];
    const estimateDirection = node[NodeProp.PLANNER_ESTIMATE_DIRECTION];
    let text = '';
    switch (estimateDirection) {
      case EstimateDirection.over:
        text += '<i class="fa fa-arrow-up"></i> over';
        break;
      case EstimateDirection.under:
        text += '<i class="fa fa-arrow-down"></i> under';
        break;
      default:
        text += 'Correctly';
    }
    text += ' estimated';
    text += estimateFactor ? ' by <b>' + factor(estimateFactor) + '</b>' : '';
    return text;
  }

  private costTooltip(node: Node): string {
    return [
      'Cost: ',
      rows(node[NodeProp.ACTUAL_COST]),
    ].join('');
  }

  private buffersTooltip(node: Node): string {
    let text = '';
    let hit;
    let read;
    let written;
    let dirtied;
    switch (this.viewOptions.buffersMetric) {
      case BuffersMetric.shared:
        hit = node[NodeProp.SHARED_HIT_BLOCKS];
        read = node[NodeProp.SHARED_READ_BLOCKS];
        dirtied = node[NodeProp.SHARED_DIRTIED_BLOCKS];
        written = node[NodeProp.SHARED_WRITTEN_BLOCKS];
        break;
      case BuffersMetric.temp:
        hit = node[NodeProp.TEMP_HIT_BLOCKS];
        read = node[NodeProp.TEMP_READ_BLOCKS];
        dirtied = node[NodeProp.TEMP_DIRTIED_BLOCKS];
        written = node[NodeProp.TEMP_WRITTEN_BLOCKS];
        break;
      case BuffersMetric.local:
        hit = node[NodeProp.LOCAL_HIT_BLOCKS];
        read = node[NodeProp.LOCAL_READ_BLOCKS];
        dirtied = node[NodeProp.LOCAL_DIRTIED_BLOCKS];
        written = node[NodeProp.LOCAL_WRITTEN_BLOCKS];
        break;
    }
    text += hit ? '<br>Hit: ' + rows(hit) : '';
    text += read ? '<br>Read: ' + rows(read) : '';
    text += dirtied ? '<br>Dirtied: ' + rows(dirtied) : '';
    text += written ? '<br>Written: ' + rows(written) : '';
    text = text ? text : ' N/A';
    switch (this.viewOptions.buffersMetric) {
      case BuffersMetric.shared:
        text = 'Shared Blocks:' + text;
        break;
      case BuffersMetric.temp:
        text = 'Temp Blocks:' + text;
        break;
      case BuffersMetric.local:
        text = 'Local Blocks:' + text;
        break;
    }
    return text;
  }

  private nodeType(row: any[]): string {
    return row[1][NodeProp.NODE_TYPE];
  }

  private flatten(output: any[], level: number, node: any, isLast: boolean,
                  branches: number[]) {
    // [level, node, isLastSibbling, branches]
    output.push([level, node, isLast, _.concat([], branches)]);
    if (!isLast) {
      branches.push(level);
    }

    _.each(node.Plans, (subnode) => {
      this.flatten(
        output,
        level + 1,
        subnode,
        subnode === _.last(node.Plans),
        branches,
      );
    });
    if (!isLast) {
      branches.pop();
    }
  }

  private get maxEstimateFactor(): number {
    const max = _.max(_.map(this.plans, (plan) => {
      return _.max(_.map(plan, (row) => {
        return row[1][NodeProp.PLANNER_ESTIMATE_FACTOR];
      }));
    }));
    return max * 2 || 1;
  }
}
</script>
