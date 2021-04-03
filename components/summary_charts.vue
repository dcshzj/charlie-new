<template lang="pug">
#summary-charts
  .summary-charts(v-for="(repo, i) in filteredRepos")
    .summary-charts__title(v-if="filterGroupSelection !== 'groupByNone'")
      .summary-charts__title--index {{ i+1 }}
      .summary-charts__title--groupname
        template(v-if="filterGroupSelection === 'groupByRepos'") {{ repo[0].repoName }}
        template(
          v-else-if="filterGroupSelection === 'groupByAuthors'",
          v-bind:class=" { warn: repo[0].name === '-' }"
        ) {{ repo[0].displayName }} ({{ repo[0].name }})
      .summary-charts__title--contribution(
        title="Total contribution of group"
      ) [{{ getGroupTotalContribution(repo) }} lines]
      a(
        v-if="!isGroupMerged(getGroupName(repo))",
        v-on:click="handleMergeGroup(getGroupName(repo))",
        title="click to merge group"
      )
        font-awesome-icon.icon-button(:icon="['fas', 'chevron-up']")
      a(
        v-if="isGroupMerged(getGroupName(repo))",
        v-on:click="handleExpandGroup(getGroupName(repo))",
        title="click to expand group"
      )
        font-awesome-icon.icon-button(:icon="['fas', 'chevron-down']")
      a(
        v-if="filterGroupSelection === 'groupByRepos'",
        v-bind:href="getRepoLink(repo[0])", target="_blank",
        title="click to view group's repo"
      )
        font-awesome-icon.icon-button(:icon="['fab', 'github']")
      a(
        v-else-if="filterGroupSelection === 'groupByAuthors'",
        v-bind:href="getAuthorProfileLink(repo[0].name)", target="_blank",
        title="click to view author's profile"
      )
        font-awesome-icon.icon-button(icon="user")
      template(v-if="isGroupMerged(getGroupName(repo))")
        a(
          v-if="filterGroupSelection !== 'groupByAuthors'",
          title="click to view group's code",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabAuthorship(repo[0], repo, 0, isGroupMerged(getGroupName(repo)))"
        )
          font-awesome-icon.icon-button(icon="code")
        a(
          title="click to view breakdown of commits",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabZoom(repo[0], filterSinceDate, filterUntilDate, isGroupMerged(getGroupName(repo)))"
        )
          font-awesome-icon.icon-button(icon="list-ul")
      .summary-charts__title--percentile {{ getPercentile(i) }} %
    .summary-charts__fileType--breakdown(v-if="filterBreakdown")
      template(v-if="filterGroupSelection !== 'groupByNone'")
        .summary-charts__fileType--breakdown__legend(
          v-for="fileType in getFileTypes(repo)",
          v-bind:key="fileType"
        )
          font-awesome-icon(
            icon="circle",
            v-bind:style="{ 'color': fileTypeColors[fileType] }"
          )
          span &nbsp; {{ fileType }} &nbsp;
    .summary-chart(v-for="(user, j) in repo")
      .summary-chart__title(v-if="!isGroupMerged(getGroupName(repo))")
        .summary-chart__title--index {{ j+1 }}
        .summary-chart__title--repo(v-if="filterGroupSelection === 'groupByNone'") {{ user.repoName }}
        .summary-chart__title--author-repo(v-if="filterGroupSelection === 'groupByAuthors'") {{ user.repoName }}
        .summary-chart__title--name(
          v-if="filterGroupSelection !== 'groupByAuthors'",
          v-bind:class="{ warn: user.name === '-' }"
        ) {{ user.displayName }} ({{ user.name }})
        .summary-chart__title--contribution.mini-font [{{ user.checkedFileTypeContribution }} lines]
        a(
          v-if="filterGroupSelection !== 'groupByRepos'",
          v-bind:href="getRepoLink(repo[j])", target="_blank",
          title="click to view repo"
        )
          font-awesome-icon.icon-button(:icon="['fab', 'github']")
        a(
          v-if="filterGroupSelection !== 'groupByAuthors'",
          v-bind:href="getAuthorProfileLink(repo[j].name)", target="_blank",
          title="click to view author's profile"
        )
          font-awesome-icon.icon-button(icon="user")
        a(
          title="click to view author's code",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabAuthorship(user, repo, j, isGroupMerged(getGroupName(repo)))"
        )
          font-awesome-icon.icon-button(icon="code")
        a(
          title="click to view breakdown of commits",
          onclick="deactivateAllOverlays()",
          v-on:click="openTabZoom(user, filterSinceDate, filterUntilDate, isGroupMerged(getGroupName(repo)))"
        )
          font-awesome-icon.icon-button(icon="list-ul")
        .summary-chart__title--percentile(
          v-if="filterGroupSelection === 'groupByNone'"
        ) {{ getPercentile(j) }} %

      .summary-chart__ramp(
        v-on:click="openTabZoomSubrange(user, $event, isGroupMerged(getGroupName(repo)))"
      )
        v-ramp(
          v-bind:groupby="filterGroupSelection",
          v-bind:user="user",
          v-bind:tframe="filterTimeFrame",
          v-bind:sdate="filterSinceDate",
          v-bind:udate="filterUntilDate",
          v-bind:avgsize="avgCommitSize",
          v-bind:mergegroup="isGroupMerged(getGroupName(repo))",
          v-bind:filtersearch="filterSearch")
        .overlay

      .summary-chart__contribution
        template(v-if="filterBreakdown")
          .summary-chart__contrib(
            v-for="(widths, fileType) in getFileTypeContributionBars(user.fileTypeContribution)"
          )
            .summary-chart__contrib--bar(
              v-for="width in widths",
              v-bind:style="{ width: width + '%',\
                'background-color': fileTypeColors[fileType] }",
              v-bind:title="fileType + ': ' + user.fileTypeContribution[fileType] + ' lines, '\
                + 'total: ' + user.checkedFileTypeContribution + ' lines ' + '(contribution from ' + minDate + ' to '\
                + maxDate + ')'"
            )
        template(v-else)
          .summary-chart__contrib(
            v-bind:title="'Total contribution from ' + minDate + ' to ' + maxDate + ': '\
              + user.checkedFileTypeContribution + ' lines'"
          )
            .summary-chart__contrib--bar(
              v-for="width in getContributionBars(user.checkedFileTypeContribution)",
              v-bind:style="{ width: width+'%' }"
            )
</template>

<script>
/* global Vuex */
export default {
  props: ['checkedFileTypes', 'filtered', 'avgContributionSize', 'filterBreakdown',
      'filterGroupSelection', 'filterTimeFrame', 'filterSinceDate', 'filterUntilDate', 'isMergeGroup',
      'minDate', 'maxDate', 'filterSearch'],
  data() {
    return {
      drags: [],
    };
  },
  computed: {
    avgCommitSize() {
      let totalCommits = 0;
      let totalCount = 0;
      this.filteredRepos.forEach((repo) => {
        repo.forEach((user) => {
          user.commits.forEach((slice) => {
            if (slice.insertions > 0) {
              totalCount += 1;
              totalCommits += slice.insertions;
            }
          });
        });
      });
      return totalCommits / totalCount;
    },

    filteredRepos() {
      return this.filtered.filter((repo) => repo.length > 0);
    },

    ...Vuex.mapState(['mergedGroups', 'fileTypeColors']),
  },
  methods: {
    getFileTypeContributionBars(fileTypeContribution) {
      let currentBarWidth = 0;
      const fullBarWidth = 100;
      const contributionPerFullBar = (this.avgContributionSize * 2);
      const allFileTypesContributionBars = {};

      Object.keys(fileTypeContribution)
          .filter((fileType) => this.checkedFileTypes.includes(fileType))
          .forEach((fileType) => {
            const contribution = fileTypeContribution[fileType];
            let barWidth = (contribution / contributionPerFullBar) * fullBarWidth;
            const contributionBars = [];

            // if contribution bar for file type is able to fit on the current line
            if (currentBarWidth + barWidth < fullBarWidth) {
              contributionBars.push(barWidth);
              currentBarWidth += barWidth;
            } else {
              // take up all the space left on the current line
              contributionBars.push(fullBarWidth - currentBarWidth);
              barWidth -= fullBarWidth - currentBarWidth;
              // additional bar width will start on a new line
              const numOfFullBars = Math.floor(barWidth / fullBarWidth);
              for (let i = 0; i < numOfFullBars; i += 1) {
                contributionBars.push(fullBarWidth);
              }
              const remainingBarWidth = barWidth % fullBarWidth;
              if (remainingBarWidth !== 0) {
                contributionBars.push(remainingBarWidth);
              }
              currentBarWidth = remainingBarWidth;
            }

            allFileTypesContributionBars[fileType] = contributionBars;
          });

      return allFileTypesContributionBars;
    },

    getFileTypes(repo) {
      const fileTypes = [];
      repo.forEach((user) => {
        Object.keys(user.fileTypeContribution).forEach((fileType) => {
          if (this.checkedFileTypes.includes(fileType) && !fileTypes.includes(fileType)) {
            fileTypes.push(fileType);
          }
        });
      });
      return fileTypes;
    },

    getGroupTotalContribution(group) {
      return group.reduce((acc, ele) => acc + ele.checkedFileTypeContribution, 0);
    },

    getContributionBars(totalContribution) {
      const res = [];
      const contributionLimit = (this.avgContributionSize * 2);

      const cnt = parseInt(totalContribution / contributionLimit, 10);
      for (let cntId = 0; cntId < cnt; cntId += 1) {
        res.push(100);
      }

      const last = (totalContribution % contributionLimit) / contributionLimit;
      if (last !== 0) {
        res.push(last * 100);
      }

      return res;
    },

    getAuthorProfileLink(userName) {
      return `https://github.com/${userName}`;
    },

    getRepoLink(repo) {
      const { REPOS } = window;
      const { location, branch } = REPOS[repo.repoId];

      if (Object.prototype.hasOwnProperty.call(location, 'organization')) {
        return `${window.BASE_URL}/${location.organization}/${location.repoName}/tree/${branch}`;
      }

      return repo.location;
    },

    // triggering opening of tabs //
    openTabAuthorship(user, repo, index, isMerged) {
      const {
        minDate, maxDate, checkedFileTypes,
      } = this;

      const info = {
        minDate,
        maxDate,
        checkedFileTypes,
        author: user.name,
        repo: user.repoName,
        name: user.displayName,
        isMergeGroup: isMerged,
        location: this.getRepoLink(repo[index]),
        repoIndex: index,
      };

      this.$store.commit('updateTabAuthorshipInfo', info);
    },

    openTabZoomSubrange(user, evt, isMerge) {
      const isKeyPressed = window.isMacintosh ? evt.metaKey : evt.ctrlKey;

      if (isKeyPressed) {
        if (this.drags.length === 0) {
          this.dragViewDown(evt);
        } else {
          this.dragViewUp(evt);
        }
      }

      // skip if accidentally clicked on ramp chart
      if (this.drags.length === 2 && this.drags[1] - this.drags[0]) {
        const tdiff = new Date(this.filterUntilDate) - new Date(this.filterSinceDate);
        const idxs = this.drags.map((x) => x * tdiff / 100);
        const tsince = window.getDateStr(new Date(this.filterSinceDate).getTime() + idxs[0]);
        const tuntil = window.getDateStr(new Date(this.filterSinceDate).getTime() + idxs[1]);
        this.drags = [];
        this.openTabZoom(user, tsince, tuntil, isMerge);
      }
    },

    openTabZoom(user, since, until, isMerge) {
      const {
        avgCommitSize, filterGroupSelection, filterTimeFrame, filterSearch,
      } = this;
      const clonedUser = Object.assign({}, user); // so that changes in summary won't affect zoom
      const info = {
        zRepo: user.repoName,
        zAuthor: user.name,
        zFilterGroup: filterGroupSelection,
        zTimeFrame: filterTimeFrame,
        zAvgCommitSize: avgCommitSize,
        zUser: clonedUser,
        zLocation: this.getRepoLink(user),
        zSince: since,
        zUntil: until,
        zIsMerge: isMerge,
        zFromRamp: false,
        zFilterSearch: filterSearch,
      };

      this.$store.commit('updateTabZoomInfo', info);
    },

    getBaseTarget(target) {
      return target.className === 'summary-chart__ramp'
          ? target
          : this.getBaseTarget(target.parentElement);
    },

    dragViewDown(evt) {
      window.deactivateAllOverlays();

      const pos = evt.clientX;
      const ramp = this.getBaseTarget(evt.target);
      this.drags = [pos];

      const base = ramp.offsetWidth;
      const offset = ramp.parentElement.offsetLeft;

      const overlay = ramp.getElementsByClassName('overlay')[0];
      overlay.style.marginLeft = '0';
      overlay.style.width = `${(pos - offset) * 100 / base}%`;
      overlay.className += ' edge';
    },

    dragViewUp(evt) {
      window.deactivateAllOverlays();
      const ramp = this.getBaseTarget(evt.target);

      const base = ramp.offsetWidth;
      this.drags.push(evt.clientX);
      this.drags.sort((a, b) => a - b);

      const offset = ramp.parentElement.offsetLeft;
      this.drags = this.drags.map((x) => (x - offset) * 100 / base);

      const overlay = ramp.getElementsByClassName('overlay')[0];
      overlay.style.marginLeft = `${this.drags[0]}%`;
      overlay.style.width = `${this.drags[1] - this.drags[0]}%`;
      overlay.className += ' show';
    },

    getPercentile(index) {
      if (this.filterGroupSelection === 'groupByNone') {
        return (Math.round((index + 1) * 1000 / this.filtered[0].length) / 10).toFixed(1);
      }
      return (Math.round((index + 1) * 1000 / this.filtered.length) / 10).toFixed(1);
    },

    getGroupName(group) {
      return window.getGroupName(group, this.filterGroupSelection);
    },

    isGroupMerged(groupName) {
      return this.mergedGroups.includes(groupName);
    },

    handleMergeGroup(groupName) {
      const info = this.mergedGroups;
      info.push(groupName);
      this.$store.commit('updateMergedGroup', info);
    },

    handleExpandGroup(groupName) {
      const info = this.mergedGroups.filter((x) => x !== groupName);
      this.$store.commit('updateMergedGroup', info);
    },
  },
  components: {
    vRamp: window.vRamp,
  },
}
</script>

<style lang="scss" scoped>
.summary-charts {
  margin-bottom: 3rem;

  &__title {
    align-items: center;
    display: flex;
    font-weight: bold;
    text-align: left;

    & > * {
      padding-right: .5rem;
    }

    &--index {
      background: mui-color('black');
      color: mui-color('white');
      font-size: medium;
      overflow: hidden;
      padding: .1em .25em;
      vertical-align: middle;
    }

    &--groupname {
      font-size: medium;
      padding: .5rem;
    }

    &--percentile {
      @include mini-font;
      color: mui-color('grey');
      margin-left: auto;
    }

    &--contribution {
      @include mini-font;
      display: inline;
    }
  }

  &__fileType--breakdown {
    overflow-y: hidden;

    &__legend {
      display: inline;
      float: left;
    }
  }
}

.summary-chart {
  margin-bottom: 1rem;
  overflow: auto;
  position: relative;
  text-align: left;

  &__title {
    align-items: center;
    clear: left;
    display: flex;

    & > * {
      padding-right: .5rem;
    }

    &--index,
    &--repo {
      font-weight: bold;
    }

    &--index::after {
      content: '.';
    }

    &--repo {
      padding-right: .25rem;
    }

    &--contribution {
      @include mini-font;
    }

    &--percentile {
      @include mini-font;
      color: mui-color('grey');
      margin-left: auto;
      padding-right: 0;
    }
  }

  &__ramp {
    position: relative;

    .overlay {
      height: 100%;
      position: absolute;
      top: 0;

      &.show {
        background-color: rgba(mui-color('white'), .5);
        border: 1px dashed mui-color('black');
      }

      &.edge {
        border-right: 1px dashed mui-color('black');
      }
    }
  }

  &__contrib {
    text-align: left;

    &--bar {
      background-color: mui-color('red');
      float: left;
      height: 4px;
      margin-top: 2px;
    }
  }
}
</style>
