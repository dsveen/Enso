<template>

    <tabs alignment="centered"
        custom>
        <span slot="label"
            slot-scope="{ tab }">
            {{ tab }}
            <span class="tag is-dark file-counter">
                {{ content(tab).length }}
            </span>
        </span>
        <div class="columns is-reverse-mobile">
            <div class="column is-two-thirds">
                <tab v-for="(folder, index) in folders"
                    :key="index"
                    :id="folder">
                    <div class="columns is-multiline is-mobile">
                        <div class="column is-half-mobile is-one-third-desktop"
                            v-for="(file, index) in content(folder)"
                            :key="index">
                            <file :file="file"
                                @delete="destroy(file.id)"/>
                        </div>
                    </div>
                    <h4 class="subtitle is-4 has-text-centered"
                        v-if="content(folder).length === 0">
                        {{ __('No files found the selected category') }}
                    </h4>
                </tab>
            </div>
            <div class="column is-one-third">
                <div class="has-margin-top-small">
                    <p class="control has-icons-left has-icons-right"
                        v-if="files.length">
                        <input class="input is-rounded search-files"
                            v-model="query">
                        <span class="icon is-small is-left">
                            <fa icon="search"/>
                        </span>
                        <span class="icon is-small is-right clear-button"
                            v-if="query && !loading">
                            <a class="delete is-small"
                                @click="query = null"/>
                        </span>
                    </p>
                </div>
                <date-interval-filter class="box has-margin-top-large"
                    :title="__('Uploaded Between')"
                    :min="interval.min"
                    @update-min="interval.min = $event; fetch()"
                    :max="interval.max"
                    @update-max="interval.max = $event; fetch()"/>
                <button class="button is-fullwidth"
                    :class="{ 'is-loading': loading }"
                    @click="fetch">
                    <span>
                        {{ __('Reload') }}
                    </span>
                    <span class="icon">
                        <fa icon="sync-alt"/>
                    </span>
                </button>
                <div class="box has-margin-top-large">
                    <h4 class="title is-4 has-text-centered">
                        {{ __('Storage used') }}: {{ totalStorage / 1000 | numberFormat }} KB
                    </h4>
                    <chart :data="chartData"
                        :options="{ aspectRatio: 1 }"
                        type="pie"/>
                </div>
            </div>
        </div>
    </tabs>

</template>

<script>

import { library } from '@fortawesome/fontawesome-svg-core';
import { faSearch, faSyncAlt } from '@fortawesome/free-solid-svg-icons';

import Tabs from '../../components/enso/bulma/Tabs.vue';
import Tab from '../../components/enso/bulma/Tab.vue';
import DateIntervalFilter from '../../components/enso/bulma/DateIntervalFilter.vue';
import File from '../../components/enso/filemanager/File.vue';
import Chart from '../../components/enso/charts/Chart.vue';

import Colors from '../../components/enso/charts/colors';

library.add(faSearch, faSyncAlt);

export default {
    components: {
        Tabs, Tab, File, DateIntervalFilter, Chart,
    },

    data() {
        return {
            loading: false,
            files: [],
            folders: [],
            query: null,
            interval: {
                min: null,
                max: null,
            },
        };
    },

    computed: {
        filteredFiles() {
            return this.query
                ? this.files.filter(file => file.name.toLowerCase()
                    .indexOf(this.query.toLowerCase()) > -1)
                : this.files;
        },
        colors() {
            return Colors.slice(0, this.folders.length);
        },
        foldersStats() {
            return this.folders.map(folder =>
                this.content(folder)
                    .reduce((total, { size }) => (total += size), 0));
        },
        chartData() {
            return {
                labels: this.folders,
                datasets: [{
                    data: this.foldersStats,
                    backgroundColor: this.colors,
                    datalabels: {
                        backgroundColor: this.colors,
                        formatter: val => `${this.$options.filters.numberFormat(val / 1000)} KB`,
                    },
                }],
            };
        },
        totalStorage() {
            return this.files.length
                ? this.files.reduce((total, { size }) => (total += size), 0)
                : 0;
        },
    },

    created() {
        this.fetch();
    },

    methods: {
        fetch() {
            this.loading = true;

            axios.get(
                route('core.files.index'),
                { params: { interval: this.interval } },
            ).then(({ data }) => {
                this.files = data.data;
                this.folders = data.types;
                this.loading = false;
            }).catch(error => this.handleErorr(error));
        },
        destroy(id) {
            this.loading = true;

            axios.delete(route('core.files.destroy', id, false)).then(() => {
                this.loading = false;
                const index = this.files.findIndex(file => file.id === id);
                this.files.splice(index, 1);
            }).catch((error) => {
                this.handleError(error);
            });
        },
        content(folder) {
            return this.filteredFiles.filter(({ type }) => type === folder);
        },
    },
};

</script>

<style lang="scss" scoped>

    input.search-files {
        width: 100%;
    }

    .control.has-icons-right .icon.clear-button {
        pointer-events: all;
    }

    .tag.file-counter {
        height: unset;
    }

</style>
