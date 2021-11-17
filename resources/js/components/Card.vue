<template>
  <card class="flex flex-col nova-detached-filters-card">
    <div class="px-3 py-4 detached-filters" :class="{ collapsed: isCollapsed }">
      <div class="flex flex-wrap" :class="getWidth(item)" v-for="item in card.filters" :key="item.key">
        <!-- Single Filter -->
        <nova-detached-filter
          v-if="isFilterComponent(item)"
          :width="'w-full'"
          :filter="item"
          :resource-name="resourceName"
          @handle-filter-changed="handleFilterChanged"
          @reset-filter="resetFilter"
        />

        <!-- Filter Column -->
        <nova-detached-filter
          v-else
          v-for="filter in item.filters"
          v-bind:key="filter.key"
          :width="getWidth(filter)"
          :filter="filter"
          :resource-name="resourceName"
          @handle-filter-changed="handleFilterChanged"
          @reset-filter="resetFilter"
        />
      </div>
    </div>
    <div class="detached-filters-buttons">
      <div class="detached-filters-button per-page-button" v-if="hasPerPageOptions">
        <select
          name="detached-per-page-select"
          slot="select"
          class="block w-full form-control-sm form-select"
          :value="currentPerPage"
          @change="perPageChanged($event)"
        >
          <option v-for="option in perPageOptions" :key="option">
            {{ option }}
          </option>
        </select>
      </div>
      <div class="detached-filters-button" v-if="card.withReset" @click="clearAllFilters()">
        <filter-icon/>
      </div>
      <div class="detached-filters-button" v-if="card.persistFilters" @click="toggleIsPersisting">
        <persist-filter-icon :class="{ active: isPersisting }"/>
      </div>
      <div class="detached-filters-button" v-if="card.withToggle" @click="toggleIsCollapsed">
        <collapse-icon :class="{ collapsed: isCollapsed }"/>
      </div>
    </div>
  </card>
</template>

<script>
import {Filterable, InteractsWithQueryString, PerPageable} from 'laravel-nova';
import FilterIcon from "./icons/FilterIcon"
import PersistFilterIcon from "./icons/PersistFilterIcon";
import CollapseIcon from "./icons/CollapseIcon";

export default {
  components: {CollapseIcon, PersistFilterIcon, FilterIcon},
  mixins: [Filterable, InteractsWithQueryString, PerPageable],
  props: ['card', 'resourceName', 'viaResource', 'viaRelationship'],
  data: () => ({
    perPageStyle: null,
    persistedFilters: JSON.parse(localStorage.getItem('PERSISTED_DETACHED_FILTERS')),
    persistedResources: JSON.parse(localStorage.getItem('PERSIST_DETACHED_FILTERS')),
    collapsedResources: JSON.parse(localStorage.getItem('COLLAPSED_DETACHED_FILTERS')),

    isPersisting: false,
    isCollapsed: false,
  }),

  created() {
    this.initialiseIsPersisting();
    this.initialiseIsCollapsed();

    if (this.isPersisting) {
      if (this.persistedFilters && this.persistedFilters[this.resourceName]) this.loadPersistedFilters();
      else this.initializePersistedFilters();
    }
  },

  mounted() {
    if (!this.card.showPerPageInMenu) this.perPageDropdownStyle(true);
  },

  destroyed() {
    if (!this.card.showPerPageInMenu) this.perPageDropdownStyle(false);
  },

  methods: {
    getWidth(filter) {
      if (filter.width) return filter.width;
      return 'w-auto';
    },

    resetFilter(filter) {
      this.$store.commit(`${this.resourceName}/updateFilterState`, {
        filterClass: filter.class,
        value: null,
      });

      this.handleFilterChanged(filter);
    },

    isFilterComponent(item) {
      return !!item.options && !!item.component;
    },

    toggleIsPersisting() {
      if (!this.persistedResources) this.persistedResources = {};
      if (this.persistedResources[this.resourceName]) this.persistedResources[this.resourceName] = [];

      this.persistedResources[this.resourceName] = !this.isPersisting;
      this.isPersisting = !this.isPersisting;

      localStorage.setItem('PERSIST_DETACHED_FILTERS', JSON.stringify(this.persistedResources));

      this.initializePersistedFilters();
      if (this.isPersisting) this.loadPersistedFromFilters();
    },

    toggleIsCollapsed() {
      if (!this.collapsedResources) this.collapsedResources = {};
      if (this.collapsedResources[this.resourceName]) this.collapsedResources[this.resourceName] = [];

      this.collapsedResources[this.resourceName] = !this.isCollapsed;
      this.isCollapsed = !this.isCollapsed;

      localStorage.setItem('COLLAPSED_DETACHED_FILTERS', JSON.stringify(this.collapsedResources));
    },

    loadPersistedFilters() {
      this.persistedFilters[this.resourceName].forEach(filterItem => {
        this.$store.commit(`${this.resourceName}/updateFilterState`, {
          filterClass: filterItem.filterClass,
          value: filterItem.value,
        });
      });

      this.filterChanged();
    },

    handleFilterChanged(filter) {
      if (this.isPersisting) {
        // Get updated filter from $store;
        const updatedFilter = this.getFilter(filter.class);
        if (!updatedFilter) return;
        // Get filter index in localStorage;
        const filterIndex = this.persistedFilters[this.resourceName].findIndex(f => filter.class === f.filterClass);
        if (filterIndex == null || filterIndex < 0 || !this.persistedFilters[this.resourceName][filterIndex]) {
          // If key-value pair doesn't exist in localStorage, save new
          this.persistedFilters[this.resourceName].push({
            filterClass: filter.class,
            value: updatedFilter.currentValue,
          });
        } else {
          // If exists, update value
          this.persistedFilters[this.resourceName][filterIndex].value = updatedFilter.currentValue;
        }

        localStorage.setItem('PERSISTED_DETACHED_FILTERS', JSON.stringify(this.persistedFilters));
      }

      this.filterChanged();
    },

    getFilter(filterKey) {
      return this.$store.getters[`${this.resourceName}/getFilter`](filterKey);
    },

    getFilters() {
      return this.$store.getters[`${this.resourceName}/filters`];
    },

    initializePersistedFilters() {
      if (!this.persistedFilters) this.persistedFilters = {};
      this.persistedFilters[this.resourceName] = [];

      localStorage.setItem('PERSISTED_DETACHED_FILTERS', JSON.stringify(this.persistedFilters));
    },

    clearAllFilters() {
      this.initializePersistedFilters();
      this.clearSelectedFilters();
    },

    /**
     * Handle a filter state change.
     */
    filterChanged() {
      const encodedFilters = this.$store.getters[`${this.resourceName}/currentEncodedFilters`];

      if (this.$route.query[this.pageParameter] !== '1' || this.$route.query[this.filterParameter] !== encodedFilters)
        this.updateQueryString({
          [this.pageParameter]: 1,
          [this.filterParameter]: encodedFilters,
        });
    },

    /**
     * Update the desired amount of resources per page.
     */
    perPageChanged(event) {
      Nova.$emit('change-per-page', event.target.value);
    },

    /**
     * Load persisted filters from existing filters
     */
    loadPersistedFromFilters() {
      this.getFilters().forEach(filterItem => {
        this.persistedFilters[this.resourceName].push({
          filterClass: filterItem.class,
          value: filterItem.currentValue,
        });
      });

      localStorage.setItem('PERSISTED_DETACHED_FILTERS', JSON.stringify(this.persistedFilters));
    },

    /**
     * Remove or add dropdown per-page filter style tag to head
     */
    perPageDropdownStyle(addStyle) {
      const css = "div[dusk='filter-per-page'] { display: none !important }";
      const head = document.head || document.getElementsByTagName('head')[0];

      if (!this.perPageStyle) this.perPageStyle = document.createElement('style');
      addStyle ? head.appendChild(this.perPageStyle) : head.removeChild(this.perPageStyle);

      if (addStyle) this.perPageStyle.appendChild(document.createTextNode(css));
    },

    initialiseIsPersisting() {
      if (!this.persistedResources || !this.persistedResources[this.resourceName])
        return (this.isPersisting = this.card.persistFiltersDefault);
      this.isPersisting = this.persistedResources[this.resourceName];
    },

    initialiseIsCollapsed() {
      if (!this.collapsedResources || !this.collapsedResources[this.resourceName]) return (this.isCollapsed = false);
      this.isCollapsed = this.collapsedResources[this.resourceName];
    },
  },

  computed: {
    pageParameter() {
      return this.viaRelationship ? this.viaRelationship + '_page' : this.resourceName + '_page';
    },

    /**
     * Resource card has per-page options
     */
    hasPerPageOptions() {
      return Array.isArray(this.perPageOptions);
    },

    /**
     * The per-page options configured for this resource.
     */
    perPageOptions() {
      return this.card.perPageOptions;
    },

    /**
     * Get the current per page value from the query string.
     */
    currentPerPage() {
      return this.$route.query[this.perPageParameter] || this.perPageOptions[0];
    },

    /**
     * Get the name of the per page query string variable.
     */
    perPageParameter() {
      return this.viaRelationship ? this.viaRelationship + '_per_page' : this.resourceName + '_per_page';
    },
  },
};
</script>

<style lang="scss">
.nova-detached-filters-card {
  $transition: cubic-bezier(0.6, 0.4, 0.1, 0.9);
  height: auto;
  position: relative;

  .detached-filters {
    display: flex;
    flex-wrap: wrap;
    max-height: 100vh;
    opacity: 1;
    z-index: 10;
    transition: all 0.5s $transition;
    will-change: max-height, transform, opacity, padding-top, padding-bottom;

    &.collapsed {
      opacity: 0;
      padding-top: 0.5rem;
      padding-bottom: 0.5rem;
      transform: translateY(-10%);
      max-height: 0;
      overflow: hidden;
      cursor: default;
    }
  }

  .detached-filter {
    height: auto;
    opacity: 1;

    > div:first-of-type {
      width: 100%;
    }

    h3 {
      background-color: transparent;
      text-transform: capitalize;
      padding: 0.25rem 4rem 0 0.5rem;
      font-size: 16px;
      font-weight: 300;
      font-family: Nunito, system-ui, BlinkMacSystemFont, -apple-system, sans-serif;

      white-space: nowrap;
      //overflow: hidden;
      text-overflow: ellipsis;
    }

    .reset-filter-btn {
      position: absolute;
      right: 0.4rem;
      top: 0.25rem;
    }
  }

  .detached-filters-buttons {
    position: absolute;
    display: flex;
    top: -2rem;
    right: 0;

    .detached-filters-button {
      padding: 0.5rem 0.6rem;
      background-color: var(--white);
      display: flex;
      align-items: center;

      svg {
        cursor: pointer;
      }

      &:last-of-type {
        border-top-right-radius: 0.25rem;
      }

      &:first-of-type {
        border-top-left-radius: 0.25rem;
      }

      &.per-page-button {
        padding: 0;

        select {
          border-radius: 0;
          border-color: transparent;

          &:focus {
            box-shadow: none;
          }
        }
      }
    }
  }

  .reset-filter-btn {
    opacity: 0.6;
    transition: opacity 0.3s $transition;
    cursor: pointer;

    &:hover {
      opacity: 1;
    }
  }

}
</style>
