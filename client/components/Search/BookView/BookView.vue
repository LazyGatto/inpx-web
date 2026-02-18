<template>
    <div class="row items-center q-my-sm no-wrap">
        <div class="row items-center">
            <div v-if="showRates || showDeleted">
                <div v-if="showRates && !book.del">
                    <div v-if="book.librate">
                        <q-knob
                            :model-value="book.librate"
                            :min="0"
                            :max="5"
                            size="18px"
                            font-size="12px"
                            :thickness="1"
                            :color="rateColor"
                            track-color="grey-4"
                            readonly
                        />

                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%" max-width="400px">
                            Оценка {{ book.librate }}
                        </q-tooltip>
                    </div>
                    <div v-else style="width: 18px" />
                </div>
                <div v-else class="row justify-center" style="width: 18px">
                    <q-icon v-if="book.del" class="la la-trash text-bold text-red" size="18px">
                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%" max-width="400px">
                            Удалено
                        </q-tooltip>
                    </q-icon>
                </div>
            </div>
        </div>

        <div class="q-ml-sm column">
            <!-- Автор (в режимах series/title/extended) -->
            <div v-if="(mode == 'series' || mode == 'title' || mode == 'extended') && bookAuthor" class="row">
                <div class="clickable2 text-green-10" @click.stop.prevent="emit('authorClick')">
                    {{ bookAuthor }}
                </div>
            </div>

            <!-- Строка 1: номер + название + кнопки действий -->
            <div class="row items-center wrap">
                <div v-if="book.serno" class="q-mr-xs">
                    {{ book.serno }}.
                </div>
                <div class="clickable2 q-mr-xs" :class="titleColor" @click.stop.prevent="emit('titleClick')">
                    {{ book.title }}
                </div>
                <div class="row items-center no-wrap">
                    <q-btn v-if="showInfo" flat round dense size="sm" icon="la la-info-circle" @click.stop.prevent="emit('bookInfo')">
                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%">Информация о книге</q-tooltip>
                    </q-btn>
                    <q-btn flat round dense size="sm" icon="la la-download" @click.stop.prevent="emit('download')">
                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%">Скачать</q-tooltip>
                    </q-btn>
                    <template v-for="(item, key) in config.external" :key="key">
                        <q-btn v-if="item.active" flat round dense size="sm" @click.stop.prevent="emit('ext-' + key)">
                            <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%">{{ item.hint || item.title || key }}</q-tooltip>
                            <span style="font-size: 11px">{{ item.title || key }}</span>
                        </q-btn>
                    </template>
                    <q-btn flat round dense size="sm" icon="la la-copy" @click.stop.prevent="emit('copyLink')">
                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%">Копировать ссылку</q-tooltip>
                    </q-btn>
                    <q-btn v-if="showReadLink" flat round dense size="sm" icon="la la-book-open" @click.stop.prevent="emit('readBook')">
                        <q-tooltip :delay="500" anchor="top middle" content-style="font-size: 80%">Читать онлайн</q-tooltip>
                    </q-btn>
                </div>
            </div>

            <!-- Строка 2: мета-данные -->
            <div class="row items-center no-wrap book-meta">
                <template v-if="(mode == 'title' || mode == 'extended') && bookSeries">
                    <div class="clickable2 q-mr-xs" @click.stop.prevent="emit('seriesClick')">{{ bookSeries }}</div>
                    <span class="meta-sep">·</span>
                </template>
                <template v-if="showGenres && book.genre">
                    <div class="q-mr-xs">{{ bookGenre }}</div>
                    <span class="meta-sep">·</span>
                </template>
                <div class="q-mr-xs">{{ bookSize }}, {{ book.ext }}</div>
                <template v-if="showDates && book.date">
                    <span class="meta-sep">·</span>
                    <div>{{ bookDate }}</div>
                </template>
            </div>

            <div v-show="showJson && mode == 'extended'">
                <pre style="font-size: 80%; white-space: pre-wrap;">{{ book }}</pre>
            </div>
        </div>
    </div>
</template>

<script>
//-----------------------------------------------------------------------------
import vueComponent from '../../vueComponent.js';

import * as utils from '../../../share/utils';

const componentOptions = {
    components: {
    },
    watch: {
        settings() {
            this.loadSettings();
        },
    }
};
class BookView {
    _options = componentOptions;
    _props = {
        book: Object,
        mode: String,
        genreMap: Object,
        showReadLink: Boolean,
        titleColor: { type: String, default: 'text-blue-10'},
    };

    showRates = true;
    showInfo = true;
    showGenres = true;
    showDeleted = false;
    showDates = false;
    showJson = false;

    created() {
        this.loadSettings();
    }

    loadSettings() {
        const settings = this.settings;

        this.showRates = settings.showRates;
        this.showInfo = settings.showInfo;
        this.showGenres = settings.showGenres;
        this.showDates = settings.showDates;
        this.showDeleted = settings.showDeleted;
        this.showJson = settings.showJson;
    }

    get config() {
        return this.$store.state.config;
    }

    get settings() {
        return this.$store.state.settings;
    }

    get bookAuthor() {
        if (this.book.author) {
            let a = this.book.author.split(',');
            return a.slice(0, 3).join(', ') + (a.length > 3 ? ' и др.' : '');
        }

        return '';
    }

    get bookSeries() {
        if (this.book.series) {
            return `Серия: ${this.book.series}`;
        }

        return '';
    }

    get bookSize() {
        let size = this.book.size/1024;
        let unit = 'KB';
        if (size > 1024) {
            size = size/1024;
            unit = 'MB';
        }
        return `${size.toFixed(0)}${unit}`;
    }

    get rateColor() {
        const rate = (this.book.librate > 5 ? 5 : this.book.librate);
        if (rate > 2)
            return `green-${(rate - 1)*2}`;
        else
            return `red-${10 - rate*2}`;
    }

    get bookGenre() {
        let result = [];
        const genre = this.book.genre.split(',');

        for (const g of genre) {
            const name = this.genreMap.get(g);
            if (name)
                result.push(name);
        }

        return result.join(' / ');
    }

    get bookDate() {
        if (!this.book.date)
            return '';

        return utils.sqlDateFormat(this.book.date);
    }

    emit(action) {
        this.$emit('bookEvent', {action, book: this.book});
    }
}

export default vueComponent(BookView);
//-----------------------------------------------------------------------------
</script>

<style scoped>
.clickable2 {
    cursor: pointer;
}

.text-ellipsis {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.flex-shrink-0 {
    flex-shrink: 0;
}

.book-meta {
    font-size: 88%;
    opacity: 0.85;
    flex-wrap: wrap;
}

.meta-sep {
    margin: 0 4px;
    color: var(--text-secondary);
    flex-shrink: 0;
}
</style>
