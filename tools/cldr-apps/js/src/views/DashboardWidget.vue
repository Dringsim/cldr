<template>
  <nav id="DashboardSection" class="halfheight">
    <p v-if="fetchErr" class="st-sad">{{ fetchErr }}</p>
    <div class="while-loading" v-if="!data && !fetchErr">
      <a-spin>
        <i>{{ loadingMessage }}</i>
      </a-spin>
    </div>
    <template v-if="data && !fetchErr">
      <header class="sidebyside-column-top">
        <button
          class="cldr-nav-btn dash-closebox"
          title="Close"
          @click="closeDashboard"
        >
          ✕
        </button>
        <span
          class="i-am-dashboard"
          :title="
            'Dashboard (locale: ' + localeName + '; coverage: ' + level + ')'
          "
          >Dashboard</span
        >
        <button
          class="cldr-nav-btn dash-reload"
          title="Reload"
          @click="reloadDashboard"
        >
          ↻
        </button>
        <span v-for="catData of data.notifications" :key="catData.category">
          <template v-if="catData.total">
            <button
              :category="catData.category"
              class="scrollto cldr-nav-btn"
              v-on:click.prevent="scrollToCategory"
              :title="describe(catData.category)"
            >
              {{ humanize(catData.category) }} ({{ catData.total }})
            </button>
            &nbsp;&nbsp;
          </template>
        </span>
        <span class="right-control">
          <input
            type="checkbox"
            title="Hide checked items"
            id="hideChecked"
            v-model="hideChecked"
          /><label for="hideChecked">hide</label>
        </span>
      </header>
      <section id="DashboardScroller" class="sidebyside-scrollable">
        <template
          v-for="catData of data.notifications"
          :key="'template-' + catData.category"
        >
          <template
            v-for="group of catData.groups"
            :key="group.section + group.page + group.header"
          >
            <template v-for="entry in group.entries">
              <p
                v-if="!(hideChecked && entry.checked)"
                :key="'dash-item-' + entry.xpstrid + '-' + catData.category"
                :id="'dash-item-' + entry.xpstrid + '-' + catData.category"
                :class="
                  'dash-' +
                  catData.category +
                  (lastClicked === entry.xpstrid + '-' + catData.category
                    ? ' last-clicked'
                    : '')
                "
              >
                <span class="dashEntry">
                  <a
                    v-bind:href="
                      getLink(
                        [locale, group.page, entry.xpstrid],
                        catData.category,
                        entry.code
                      )
                    "
                    @click="
                      () =>
                        setLastClicked(entry.xpstrid + '-' + catData.category)
                    "
                  >
                    <span
                      class="category"
                      :title="describe(catData.category)"
                      >{{ abbreviate(catData.category) }}</span
                    >
                    <span class="section-page" title="section—page">{{
                      humanize(group.section + "—" + group.page)
                    }}</span>
                    |
                    <span
                      v-if="group.header"
                      class="entry-header"
                      title="entry header"
                      >{{ group.header }}</span
                    >
                    |
                    <span class="code" title="code">{{ entry.code }}</span>
                    |
                    <cldr-value
                      class="previous-english"
                      title="previous English"
                      lang="en"
                      dir="ltr"
                      v-if="entry.previousEnglish"
                      >{{ entry.previousEnglish }} →</cldr-value
                    >
                    <cldr-value
                      class="english"
                      lang="en"
                      dir="ltr"
                      title="English"
                      v-if="entry.english"
                      >{{ entry.english }}</cldr-value
                    >
                    |
                    <cldr-value
                      v-if="entry.winning"
                      class="winning"
                      title="Winning"
                      >{{ entry.winning }}</cldr-value
                    >
                    <template v-if="entry.comment">
                      |
                      <span v-html="entry.comment" title="comment"></span>
                    </template>
                    <span v-if="catData.category === 'Reports'"
                      >{{ humanizeReport(entry.code) }} Report</span
                    >
                  </a>
                </span>
                <input
                  v-if="canBeHidden(catData.category)"
                  type="checkbox"
                  class="right-control"
                  title="You can hide checked items with the hide checkbox above"
                  v-model="entry.checked"
                  @change="
                    (event) => {
                      entryCheckmarkChanged(event, entry);
                    }
                  "
                />
              </p>
            </template>
          </template>
        </template>
        <p class="bottom-padding">...</p>
      </section>
    </template>
  </nav>
</template>

<script>
import * as cldrAjax from "../esm/cldrAjax.mjs";
import * as cldrCoverage from "../esm/cldrCoverage.mjs";
import * as cldrDash from "../esm/cldrDash.mjs";
import * as cldrGui from "../esm/cldrGui.mjs";
import * as cldrLoad from "../esm/cldrLoad.mjs";
import * as cldrReport from "../esm/cldrReport.mjs";
import * as cldrStatus from "../esm/cldrStatus.mjs";
import * as cldrText from "../esm/cldrText.mjs";

export default {
  props: [],
  data() {
    return {
      data: null,
      fetchErr: null,
      hideChecked: false,
      lastClicked: null,
      loadingMessage: "Loading Dashboard…",
      locale: null,
      localeName: null,
      level: null,
    };
  },

  created() {
    this.fetchData();
  },

  methods: {
    getLink(array, category, code) {
      const [locale, page, xpstrid] = array;
      if (category === "Reports") {
        return `#r_${code}/${locale}`;
      } else {
        return "#/" + array.join("/");
      }
    },
    scrollToCategory(event) {
      const whence = event.target.getAttribute("category");
      if (this.data && this.data.notifications) {
        for (let catData of this.data.notifications) {
          if (catData.category == whence) {
            const whither = document.querySelector(".dash-" + whence);
            if (whither) {
              whither.scrollIntoView(true);
            }
            return;
          }
        }
      }
    },

    reopen() {
      if (
        cldrStatus.getCurrentLocale() !== this.locale ||
        cldrCoverage.effectiveName(this.locale) !== this.level
      ) {
        this.reloadDashboard();
      }
    },

    handleCoverageChanged(level) {
      if (this.level !== level) {
        this.reloadDashboard();
      }
    },

    reloadDashboard() {
      this.data = null;
      this.fetchData();
    },

    fetchData() {
      if (!cldrStatus.getSurveyUser()) {
        this.fetchErr = "Please log in to see the Dashboard.";
        return;
      }
      this.locale = cldrStatus.getCurrentLocale();
      this.level = cldrCoverage.effectiveName(this.locale);
      if (!this.locale || !this.level) {
        this.fetchErr = "Please choose a locale and a coverage level first.";
        return;
      }
      this.localeName = cldrLoad.getLocaleName(this.locale);
      this.loadingMessage = `Loading ${this.localeName} dashboard at ${this.level} level`;
      this.reallyFetch();
    },

    reallyFetch() {
      const url = `api/summary/dashboard/${this.locale}/${this.level}`;
      cldrAjax
        .doFetch(url)
        .then((response) => {
          if (!response.ok) {
            throw new Error(response.statusText);
          }
          return response;
        })
        .then((data) => data.json())
        // hide items that TC does not need
        .then((data) => {
          const { userIsTC } = cldrStatus.getPermissions();
          if (userIsTC) {
            data.notifications = data.notifications.filter(
              // skip this category for TC users
              ({ category }) => category !== "Abstained"
            );
          }
          return data;
        })
        .then((data) => {
          this.data = cldrDash.setData(data);
          this.resetScrolling();
        })
        .catch((err) => {
          const msg = "Error loading Dashboard data: " + err;
          console.error(msg);
          this.fetchErr = msg;
        });
    },

    /**
     * A user has voted. Update the Dashboard as needed.
     *
     * @param json - the response to a request by cldrTable.refreshSingleRow
     */
    updatePath(json) {
      cldrDash.updatePath(this.data, json);
    },

    resetScrolling() {
      setTimeout(function () {
        const el = document.getElementById("DashboardScroller");
        if (el) {
          el.scrollTo(0, 0);
        }
      }, 500 /* half a second */);
    },

    setLastClicked(id) {
      this.lastClicked = id;
    },

    closeDashboard() {
      cldrGui.hideDashboard();
    },

    abbreviate(category) {
      // The category is like "English_Changed"; also allow "English Changed" with space not underscore
      if (category.toLowerCase().replaceAll(" ", "_") === "english_changed") {
        return "EC";
      } else {
        return category.substr(0, 1); // first letter, like "E" for "Error"
      }
    },

    describe(category) {
      // The category is like "English_Changed" or "English Changed"
      // The corresponding key is like "notification_category_english_changed"
      const key =
        "notification_category_" + category.toLowerCase().replaceAll(" ", "_");
      let description = cldrText.get(key);
      if (description === key) {
        console.error(
          "Dashboard is missing a description for the category: " + category
        );
        description = this.humanize(category);
      }
      return description;
    },

    humanize(str) {
      // For categories like "English_Changed", page names like "Languages_K_N",
      // and section names like "Locale_Display_Names"
      return str.replaceAll("_", " ");
    },

    humanizeReport(report) {
      return cldrReport.reportName(report);
    },

    entryCheckmarkChanged(event, entry) {
      cldrDash.saveEntryCheckmark(event.target.checked, entry, this.locale);
    },

    canBeHidden(category) {
      if (category === "Error" || category === "Missing") {
        return false;
      }
      return true;
    },
  },

  computed: {
    console: () => console,
  },
};
</script>

<style scoped>
.st-sad {
  font-style: italic;
  border: 1px dashed red;
  color: darkred;
}

#DashboardSection {
  border-top: 4px solid #cfeaf8;
  font-size: small;
}

.while-loading {
  padding-top: 3em;
}

header {
  width: 100%;
  display: flex;
  flex-wrap: wrap;
  text-align: baseline;
  align-items: center;
  margin: 0;
  padding: 1ex 0;
  background-color: white;
  background-image: linear-gradient(white, #e7f7ff);
}

.dash-closebox {
  margin-left: 1ex;
}

.i-am-dashboard {
  font-weight: bold;
  margin-right: 1ex;
}

.dash-reload {
  margin-right: 2em;
}

p {
  padding: 0.1em;
  margin: 0;
  line-height: 1.75;
  display: flex;
}

p.bottom-padding {
  line-height: 5;
  color: white;
}

.dashEntry {
  display: flex;
  width: 100%;
  flex-direction: row;
  text-align: left;
  align-items: baseline;
}

.category {
  border: 0.1em solid;
  border-radius: 50%;
  height: 100%;
  padding: 0.1em 0.33em;
  margin-right: 0.5em;
  font-weight: bold;
}

.right-control {
  /* This element will be pushed to the right.
     The elements to the left of it will be pushed to the left. */
  margin-left: auto !important;
  margin-right: 1ex;
}

.section-page {
  font-weight: bold;
}

.entry-header {
  font-weight: bold;
}

.code {
  font: bold small "Courier New", Courier, mono;
}

.english {
  color: blue;
}

.previous-english {
  color: #900;
}

.winning {
  color: green;
}

a {
  /* enable clicking on space to right of text, as well as on text itself */
  display: block;
  width: 100%;
  color: inherit;
}

a:hover {
  background-color: #ccdfff; /* light blue */
}

.last-clicked {
  background-color: #eee; /* light gray */
}
</style>
