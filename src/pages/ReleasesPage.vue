<template>
  <div>
    <div>
      <div class="container releases">
        <div class="row" v-if="state.rateLimit">
          <div class="col">
            <pre>
Last updated: {{state.lastCheckedReleases | unixtimePretty}}
Starred repos: {{state.starredReposNumber}}
Rate limit remaining: {{state.rateLimit.remaining}}
Rate limit reset: {{state.rateLimit.reset | unixtimePretty}}
            </pre>
          </div>
        </div>

        <div v-if="false">
          dayFirst={{dayFirst}}, dayLast={{dayLast}}
          <div v-for="d in days" v-if="false">{{d}} {{(releasesByDay[d] || []).length}}</div>
        </div>

        <div class="tableRow" style="margin: -10px -16px 0;">
          <template v-for="day in days">
            <table v-if="(releasesByDay[day] || []).length" border="0" cellspacing="0" cellpadding="6" class="table1">
              <tr>
                <td colspan="3" style="padding-left: 66px;">{{ day }}</td>
              </tr>
            </table>

            <table border="0" cellspacing="0" cellpadding="6" class="table1">
            <template v-for="r in releasesByDay[day]">
              <tr class="mainTr" @click="toggleClick(r.id)">
                <td style="width: 66px; padding: 10px 0 10px 12px; vertical-align: top;">
                  <img :src="r.avatarUrl" style="width: 40px; height: 40px;">
                </td>
                <td style="vertical-align: top; padding: 8px 0 0;">
                  {{r.repoFullName}}<br>
                  <span class="ver">{{r.v}}</span>
                </td>
                <td style="width: 80px; text-align: right; vertical-align: top; padding-top: 7px;">
                  {{r.published | timeHM}}
                  <md-icon style="opacity: 0.4" v-if="expandedRows.has(r.id)">expand_less</md-icon>
                  <md-icon style="opacity: 0.4" v-else>expand_more</md-icon>
                </td>
              </tr>

              <transition name="slide">
                <tr v-if="expandedRows.has(r.id)" @click="toggleClick(r.id)">
                  <td colspan="3" style="padding: 0 10px 10px 16px; word-wrap: break-word;">
                    <div>
                      <md-button
                        class="md-dense md-primary1 md-raised"
                        style="margin-left: -4px; margin-top: 10px;"
                        :href="`https://github.com/${r.repoFullName}/releases/tag/${r.tagName || 'v' + r.v}`" target="_blank" rel="noopener"
                      >
                        view on github
                      </md-button>
                    </div>

                    <div class="md" v-html="r.descr"></div>
                  </td>
                </tr>
              </transition>
            </template>
            </table>
          </template>

          <table v-if="!dayLoading && !$store.getters.getReleasesCount()" border="0" cellspacing="0" cellpadding="6" class="table1">
            <tr >
              <td colspan="3">You have 0 releases in last 30 days. Either you have too few starred projects or maybe there's a
                glitch in the system, so check back in 10 minutes.
              </td>
            </tr>
          </table>

          <table border="0" cellspacing="0" cellpadding="6" class="table1">
            <tr v-if="dayLoading">
              <td colspan="3">loading {{ dayLoading }}...</td>
            </tr>
            <tr v-else>
              <td colspan="3"><md-button class="md-raised md-primary" @click="loadMore()">load more...</md-button></td>
            </tr>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { DateTime } from 'luxon'
import Vue from 'vue'
import Component from 'vue-class-component'
import { Progress } from '../decorators/progress.decorator'
import { FeedResp, Release, ReleasesByDay, releasesService } from '../srv/releases.service'
import { GlobalState, st, store } from '../store'
import { promiseUtil } from '../util/promise.util'
import { LUXON_ISO_DATE_FORMAT, timeUtil } from '../util/time.util'

@Component
export default class ReleasesPage extends Vue {
  expandedRows = new Set<string>()
  // days: string[] = []
  maxReleases = 30
  dayFirst = ''
  dayLast = ''
  dayLoading = ''
  dayMax = ''
  releasesByDay: ReleasesByDay = {}

  get dayNext (): string {
    if (!this.dayLast) return ''
    return DateTime.fromISO(this.dayLast, {zone: 'utc'})
      .minus({days: 1})
      .toFormat(LUXON_ISO_DATE_FORMAT)
  }

  get state (): GlobalState {
    return this.$store.state
  }

  /*get releasesByDay (): ReleasesByDay {
    return store.getters.getReleasesByDay()
  }*/

  get days (): string[] {
    const days: string[] = []
    if (!this.dayFirst || !this.dayLast) return []

    for (
      let day = DateTime.fromISO(this.dayFirst, {zone: 'utc'});
      day.toFormat(LUXON_ISO_DATE_FORMAT) >= this.dayLast;
      day = day.minus({days: 1})
    ) {
      days.push(day.toFormat(LUXON_ISO_DATE_FORMAT))
    }

    return days
  }

  @Progress()
  async mounted () {
    this.maxReleases = 30
    const today = DateTime.utc()
    const todayStr = today.toFormat(LUXON_ISO_DATE_FORMAT)
    this.dayMax = today.minus({days: 30}).toFormat(LUXON_ISO_DATE_FORMAT)
    this.releasesByDay = store.getters.getReleasesByDay()
    this.dayFirst = todayStr
    this.dayLast = store.getters.getReleasesLastDay() || todayStr

    await promiseUtil.delay(1000) // give time for animations to finish
    this.dayLast = await this.loadDay(today, 0)
    this.dayLoading = ''
    // console.log('dayLast end: ' + this.dayLast)

    // cleanAfterLastDay
    store.commit('cleanAfterLastDay', this.dayLast)
  }

  private async loadDay (day: DateTime, loaded: number): Promise<string> {
    const dayStr = day.toFormat(LUXON_ISO_DATE_FORMAT)
    this.dayLoading = dayStr
    const nextDay = day.plus({days: 1})
    const nextDayStr = nextDay.toFormat(LUXON_ISO_DATE_FORMAT)
    // this.loading = 'loading...'
    // _this.dayLast = dayStr
    this.dayLast = store.getters.getReleasesLastDay()
    // console.log('dayLast: ' + this.dayLast)

    const feedResp = await releasesService.fetchReleases(dayStr, nextDayStr)
    this.releasesByDay = store.getters.getReleasesByDay()
    // this.loading = ''
    // const releasesCount = Object.keys(st().releases).length
    loaded += feedResp.releases.length
    // console.log('loaded: ' + loaded)

    if (loaded < this.maxReleases && dayStr > this.dayMax) {
      const yesterday = day.minus({days: 1})
      return this.loadDay(yesterday, loaded)
    }

    return dayStr
  }

  async loadMore () {
    const releasesCount = Object.keys(st().releases).length
    this.maxReleases = releasesCount + 30
    this.dayMax = DateTime.fromISO(this.dayMax, {zone: 'utc'}).minus({days: 30}).toFormat(LUXON_ISO_DATE_FORMAT)
    const dayNext = DateTime.fromISO(this.dayLast, {zone: 'utc'}).minus({days: 1})
    this.dayLast = await this.loadDay(dayNext, releasesCount)
    this.dayLoading = ''
  }

  toggleClick (id: string) {
    if (this.expandedRows.has(id)) {
      this.expandedRows.delete(id)
    } else {
      this.expandedRows.add(id)
    }
    this.expandedRows = new Set(this.expandedRows)
  }
}
</script>

<style lang="scss" scoped>
  @import "../scss/var";

  .releases {
    // padding-top: 20px;
  }

  .ver {
    font-family: "Courier New";
    font-size: 12px;
    font-weight: bold;
    color: #888;
    line-height: 1;
  }

  .mainTr {
    transition: all .3s ease-out;

    #{$active} {
      background-color: rgba(0, 0, 0, 0.05);
      cursor: pointer;
      transition: all .1s ease-in;
    }
  }

  .table1 {
    width: 100% !important;
    max-width: 500px;
    table-layout: fixed;
    background-color1: pink;
  }

  @media (max-width: 800px) {
    .titleRow h1 {
      // color: pink;
      font-size: 24px;
    }
    .titleRow img {
      width: 40px; height: 40px;
    }
  }

  @media (max-width: 500px) {
    .titleRow h1 {
      // color: pink;
      font-size: 18px;
    }
    .titleRow img {
      width: 40px; height: 40px;
    }

    .descrRow {
      font-size: 15px;
    }
  }

</style>
