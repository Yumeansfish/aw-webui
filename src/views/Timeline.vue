<template lang="pug">
div
  h2 Timeline

  input-timeinterval(
    v-model="daterange"
    :defaultDuration="timeintervalDefaultDuration"
    :maxDuration="maxDuration"
  ).mb-3

  // blocks
  div.d-inline-block.border.rounded.p-2.mr-2
    | Events shown:  {{ num_events }}
  div.d-inline-block.border.rounded.p-2.mr-2
    | Swimlanes:
    select(v-model="swimlane")
      option(:value='null') None
      option(value='category') Categories
      option(value='bucketType') Bucket Specific

  details.d-inline-block.bg-light.small.border.rounded.mr-2.px-2
    summary.p-2
      b Filters: {{ filter_summary }}
    div.p-2.bg-light
      table
        tr
          th.pt-2.pr-3
            label Host:
          td
            select(v-model="filter_hostname")
              option(:value='null') All
              option(v-for="host in hosts", :value="host") {{ host }}
        tr
          th.pt-2.pr-3
            label Client:
          td
            select(v-model="filter_client")
              option(:value='null') All
              option(v-for="client in clients", :value="client") {{ client }}

  div.d-inline-block.border.rounded.p-2.mr-2(v-if="num_events !== 0")
    | Events shown: {{ num_events }}
  b-alert.d-inline-block.p-2.mb-0.mt-2(v-if="num_events === 0", variant="warning", show)
    | No events match selected criteria. Timeline is not updated.

  div.float-right.small.text-muted.pt-3
    tr
      th.pt-2.pr-3
        label Duration:
      td
        select(v-model="filter_duration")
          option(:value='null') All
          option(:value='2') 2+ secs
          option(:value='5') 5+ secs
          option(:value='10') 10+ secs
          option(:value='30') 30+ sec
          option(:value='1 * 60') 1+ mins
          option(:value='2 * 60') 2+ mins
          option(:value='3 * 60') 3+ mins
          option(:value='10 * 60') 10+ mins
          option(:value='30 * 60') 30+ mins
          option(:value='1 * 60 * 60') 1+ hrs
          option(:value='2 * 60 * 60') 2+ hrs

  div(style="float: right; color: #999").d-inline-block.pt-3
    | Drag to pan and scroll to zoom

  div(v-if="buckets !== null")
    div(style="clear: both")
    vis-timeline(
      :buckets="buckets"
      :showRowLabels="true"
      :queriedInterval="daterange"
      :swimlane="swimlane"
      :updateTimelineWindow="updateTimelineWindow"
    )

    aw-devonly(reason="Not ready for production, still experimenting")
      aw-calendar(:buckets="buckets")

  div(v-else)
    h1.aw-loading Loading...
</template>

<script lang="ts">
import _ from 'lodash';
import moment from 'moment';                      
import { useSettingsStore } from '~/stores/settings';
import { useBucketsStore } from '~/stores/buckets';
import { seconds_to_duration } from '~/util/time';

export default {
  name: 'Timeline',
  data() {
    return {
      all_buckets: null as any,
      hosts: null as string[] | null,
      buckets: null as any[] | null,
      clients: null as string[] | null,
      daterange: null as [moment.Moment, moment.Moment] | null,
      maxDuration: 31 * 24 * 60 * 60,
      filter_hostname: null as string | null,
      filter_client: null as string | null,
      filter_duration: null as number | null,
      swimlane: null as string | null,
      updateTimelineWindow: true,
    };
  },
  created() {

    let query = window.location.search;  
    if (!query) {
      const hash = window.location.hash; 
      const idx  = hash.indexOf('?');
      if (idx !== -1) {
        query = hash.slice(idx);         
      }
    }
    const params = new URLSearchParams(query);
    const start  = params.get('start');
    const end    = params.get('end');
    if (start && end) {
      this.daterange = [ moment(start), moment(end) ];
    }
  },
  computed: {
    timeintervalDefaultDuration() {
      const settingsStore = useSettingsStore();
      return Number(settingsStore.durationDefault);
    },
    num_events() {
      return _.sumBy(this.buckets || [], 'events.length');
    },
    filter_summary() {
      const desc: string[] = [];
      if (this.filter_hostname)  desc.push(this.filter_hostname);
      if (this.filter_client)    desc.push(this.filter_client);
      if (this.filter_duration! > 0) desc.push(seconds_to_duration(this.filter_duration!));
      return desc.length > 0 ? desc.join(', ') : 'none';
    },
  },
  watch: {
    daterange() {
      this.updateTimelineWindow = true;
      this.getBuckets();
    },
    filter_hostname() {
      this.updateTimelineWindow = false;
      this.getBuckets();
    },
    filter_client() {
      this.updateTimelineWindow = false;
      this.getBuckets();
    },
    filter_duration() {
      this.updateTimelineWindow = false;
      this.getBuckets();
    },
    swimlane() {
      this.updateTimelineWindow = false;
      this.getBuckets();
    },
  },
  methods: {
    async getBuckets() {
      if (!this.daterange) return;
      this.all_buckets = Object.freeze(
        await useBucketsStore().getBucketsWithEvents({
          start: this.daterange[0].format(),
          end:   this.daterange[1].format(),
        })
      );
      this.hosts = Array.from(new Set(this.all_buckets.map((b: any) => b.hostname)));
      this.clients = Array.from(new Set(this.all_buckets.map((b: any) => b.client)));
      let bs = this.all_buckets;
      if (this.filter_hostname) bs = bs.filter((b: any) => b.hostname === this.filter_hostname);
      if (this.filter_client)   bs = bs.filter((b: any) => b.client   === this.filter_client);
      if (this.filter_duration! > 0) {
        bs.forEach((b: any) => {
          b.events = b.events.filter((e: any) => e.duration >= this.filter_duration!);
        });
      }
      this.buckets = bs;
    },
  },
};
</script>

<style scoped>
details {
  position: relative;
}
details[open] summary ~ * {
  visibility: visible;
  position: absolute;
  border: 1px solid #ddd;
  border-radius: 5px;
  left: 0;
  top: 2.7em;
  background: white;
  z-index: 100;
}
</style>

