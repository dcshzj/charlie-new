<template lang="pug">
#zoom
  .panel-title
    span Commits Panel
  .toolbar--multiline(v-if="filteredUser.commits.length && totalCommitMessageBodyCount")
    a(
      v-if="expandedCommitMessagesCount < totalCommitMessageBodyCount",
      v-on:click="toggleAllCommitMessagesBody(true)"
    ) show all commit messages
    a(
      v-if="expandedCommitMessagesCount > 0",
      v-on:click="toggleAllCommitMessagesBody(false)"
    ) hide all commit messages
  .panel-heading
    .group-name
      span(
        v-if="info.zFilterGroup === 'groupByAuthors'"
      ) {{ filteredUser.displayName }} ({{ filteredUser.name }})
      a(
        v-else,
        v-bind:href="info.zLocation", target="_blank",
        v-bind:title="'Click to open the repo'"
      )
        span {{ filteredUser.repoName }}
    .author(v-if="!info.zIsMerge")
      span &#8627; &nbsp;
      span(v-if="info.zFilterGroup === 'groupByAuthors'") {{ filteredUser.repoName }}
      span(v-else) {{ filteredUser.displayName }} ({{ filteredUser.name }})
    .period
      span &#8627; &nbsp;
      span {{ info.zSince }} to {{ info.zUntil }} &nbsp;
      a(v-on:click="openSummary") [Show ramp chart for this period]
  .zoom__title
    .zoom__title--granularity granularity: one ramp per {{ info.zTimeFrame }}
    .zoom__title--tags
      template(v-for="commit in filteredUser.commits")
        template(v-for="commitResult in commit.commitResults")
          template(v-if="commitResult.tags")
            .tag(
              v-for="tag in commitResult.tags",
              v-on:click="scrollToCommit(tag, `tag ${commitResult.hash}`)"
            )
              font-awesome-icon(icon="tags")
              span &nbsp;{{ tag }}

  v-ramp(
    v-bind:groupby="info.zFilterGroup",
    v-bind:user="filteredUser",
    v-bind:tframe="info.zTimeFrame",
    v-bind:sdate="info.zSince",
    v-bind:udate="info.zUntil",
    v-bind:avgsize="info.zAvgCommitSize",
    v-bind:mergegroup="info.zIsMerge",
    v-bind:fromramp="info.zFromRamp")

  .sorting.mui-form--inline
    .mui-select
      select(v-model="commitsSortType")
        option(value="time") Time
        option(value="lineOfCode") LoC
      label sort by
    .mui-select
      select(v-model="toReverseSortedCommits")
        option(v-bind:value='true') Descending
        option(v-bind:value='false') Ascending
      label order

  .zoom__day(v-for="day in filteredUser.commits", v-bind:key="day.date")
    h3(v-if="info.zTimeFrame === 'week'") Week of {{ day.date }}
    h3(v-else) {{ day.date }}
    template
      //- use tabindex to enable focus property on div
      .commit-message(
        tabindex="-1",
        v-for="slice in day.commitResults",
        v-bind:key="slice.hash",
        v-bind:class="{ 'message-body active': slice.messageBody !== '' }"
      )
        a.message-title(v-bind:href="getSliceLink(slice)", target="_blank")
          .within-border {{ slice.messageTitle.substr(0, 50) }}
          .not-within-border(v-if="slice.messageTitle.length > 50") {{ slice.messageTitle.substr(50) }}
        span &nbsp; ({{ slice.insertions }} lines) &nbsp;
        template(v-if="slice.tags", v-for="tag in slice.tags")
          .tag(tabindex="-1", v-bind:class="[`${slice.hash}`, tag]")
            font-awesome-icon(icon="tags")
            span &nbsp;{{ tag }}
        a(
          v-if="slice.messageBody !== ''",
          v-on:click="updateExpandedCommitMessagesCount",
          onclick="toggleNext(this)",
          title="Click to show/hide the commit message body"
        )
          font-awesome-icon.commit-message--button(icon="ellipsis-h")
        .body(v-if="slice.messageBody !== ''")
          pre {{ slice.messageBody }}
            .dashed-border
</template>

<script>
export default {
  props: ['info'],
  data() {
    return {
      showAllCommitMessageBody: true,
      expandedCommitMessagesCount: this.totalCommitMessageBodyCount,
      commitsSortType: 'time',
      toReverseSortedCommits: true,
    };
  },

  computed: {
    sortingFunction() {
      const commitSortFunction = this.commitsSortType === 'time'
        ? (commit) => commit.date
        : (commit) => commit.insertions;

      return (a, b) => (this.toReverseSortedCommits ? -1 : 1)
        * window.comparator(commitSortFunction)(a, b);
    },
    filteredUser() {
      const {
        zUser, zSince, zUntil, zTimeFrame,
      } = this.info;
      const filteredUser = Object.assign({}, zUser);

      const date = zTimeFrame === 'week' ? 'endDate' : 'date';
      filteredUser.commits = zUser.commits.filter(
          (commit) => commit[date] >= zSince && commit[date] <= zUntil,
      ).sort(this.sortingFunction);

      return filteredUser;
    },
    totalCommitMessageBodyCount() {
      let nonEmptyCommitMessageCount = 0;
      this.filteredUser.commits.forEach((commit) => {
        commit.commitResults.forEach((commitResult) => {
          if (commitResult.messageBody !== '') {
            nonEmptyCommitMessageCount += 1;
          }
        });
      });

      return nonEmptyCommitMessageCount;
    },
  },
  methods: {
    initiate() {
      if (!this.info.zUser) { // restoring zoom tab from reloaded page
        this.restoreZoomTab();
      }
    },

    openSummary() {
      const info = { since: this.info.zSince, until: this.info.zUntil };
      this.$store.commit('updateSummaryDates', info);
    },

    getSliceLink(slice) {
      if (this.info.zIsMerge) {
        return `${window.getBaseLink(slice.repoId)}/commit/${slice.hash}`;
      }
      return `${window.getBaseLink(this.info.zUser.repoId)}/commit/${slice.hash}`;
    },

    scrollToCommit(tag, commit) {
      const el = this.$el.getElementsByClassName(`${commit} ${tag}`)[0];
      if (el) {
        el.focus();
      }
    },

    restoreZoomTab() {
      // restore selected user's commits from v_summary
      this.$root.$emit('restoreCommits', this.info);
    },

    setInfoHash() {
      const { addHash, encodeHash } = window;
      const {
        zAvgCommitSize, zSince, zUntil, zFilterGroup,
        zTimeFrame, zIsMerge, zAuthor, zRepo, zFromRamp, zFilterSearch,
      } = this.info;

      addHash('zA', zAuthor);
      addHash('zR', zRepo);
      addHash('zACS', zAvgCommitSize);
      addHash('zS', zSince);
      addHash('zFS', zFilterSearch);
      addHash('zU', zUntil);
      addHash('zMG', zIsMerge);
      addHash('zFTF', zTimeFrame);
      addHash('zFGS', zFilterGroup);
      addHash('zFR', zFromRamp);
      encodeHash();
    },

    toggleAllCommitMessagesBody(isActive) {
      this.showAllCommitMessageBody = isActive;

      const toRename = this.showAllCommitMessageBody ? 'commit-message message-body active' : 'commit-message message-body';

      const commitMessageClasses = document.getElementsByClassName('commit-message message-body');
      Array.from(commitMessageClasses).forEach((commitMessageClass) => {
        commitMessageClass.className = toRename;
      });

      this.expandedCommitMessagesCount = isActive ? this.totalCommitMessageBodyCount : 0;
    },

    updateExpandedCommitMessagesCount() {
      this.expandedCommitMessagesCount = document.getElementsByClassName('commit-message message-body active')
          .length;
    },

    removeZoomHashes() {
      window.removeHash('zA');
      window.removeHash('zR');
      window.removeHash('zFS');
      window.removeHash('zACS');
      window.removeHash('zS');
      window.removeHash('zU');
      window.removeHash('zFGS');
      window.removeHash('zFTF');
      window.removeHash('zMG');
      window.encodeHash();
    },
  },
  created() {
    this.initiate();
  },
  mounted() {
    this.setInfoHash();
    this.updateExpandedCommitMessagesCount();
  },
  beforeDestroy() {
    this.removeZoomHashes();
  },
  components: {
    vRamp: window.vRamp,
  },
}
</script>

<style lang="scss" scoped>
@import '../assets/css/colors';

.zoom {
  &__title {
    &--granularity {
      @include mini-font;
      margin-top: .5rem;
    }

    &--tags {
      margin: .25rem 0 .25rem 0;

      .tag {
        cursor: pointer;
      }
    }
  }

  &__toggle-commit-message-body {
    padding-top: 10px;
  }

  &__day,
  &__title {
    /* Tags in commits */
    .tag {
      @include mini-font;
      background: mui-color('grey', '600');
      border-radius: 5px;
      color: mui-color('white');
      display: inline-block;
      margin: .2rem .2rem .2rem 0;
      padding: 0 3px 0 3px;

      .fa-tags {
        width: .65rem;
      }
    }
  }
}

/* Commit Message Body in Zoom Tab */
.commit-message {
  padding: 5px;

  &:focus,
  &:focus-within {
    border: 1px solid mui-color('blue', '500');
  }

  &.active {
    .body {
      background-color: mui-color('white');
      border: 1px solid mui-color('grey', '700');
      display: grid;
      margin: .25rem 0 .25rem 0;
      overflow-x: auto;
      padding: .4rem;
      resize: none;

      pre {
        position: relative;

        .dashed-border {
          border-right: 1px dashed mui-color('grey', '500'); // 72nd character line
          height: 100%;
          pointer-events: none;
          position: absolute;
          top: 0;
          width: 72ch;
        }
      }
    }
  }

  .body {
    display: none;
  }

  .tag {
    cursor: pointer;

    &:focus {
      border: 1px solid mui-color('blue', '500');
      outline: none;
    }
  }

  &--button {
    color: mui-color('grey');
    padding-left: .5rem;

    &:hover {
      cursor: pointer;
    }
  }

  pre {
    margin: 0;
  }

  span.loc {
    color: mui-color('grey');
  }

  .message-title {
    display: inline;
    font-family: monospace;

    .within-border {
      display: inline;
    }

    .not-within-border {
      border-left: 1px dashed mui-color('grey', '500'); // 50th character line
      display: inline;
    }
  }
}
</style>
