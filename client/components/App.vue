<template>
    <div class="fit row">
        <Api ref="api" v-model="accessGranted" />
        <Notify ref="notify" />
        <StdDialog ref="stdDialog" />

        <router-view v-if="accessGranted" v-slot="{ Component }">
            <keep-alive>
                <component :is="Component" class="col" />
            </keep-alive>
        </router-view>        
    </div>
</template>

<script>
//-----------------------------------------------------------------------------
import vueComponent from './vueComponent.js';

//import * as utils from '../share/utils';
import Notify from './share/Notify.vue';
import StdDialog from './share/StdDialog.vue';

import Api from './Api/Api.vue';
import Search from './Search/Search.vue';

const componentOptions = {
    components: {
        Api,
        Notify,
        StdDialog,

        Search,
    },
    watch: {
        'settings.colorScheme'(newValue) {
            this.applyColorScheme(newValue);
        },
    },

};
class App {
    _options = componentOptions;
    accessGranted = false;

    _mediaQuery = null;
    _mediaQueryHandler = null;

    created() {
        this.commit = this.$store.commit;

        this.applyColorScheme(this.settings.colorScheme || 'system');

        //root route
        let cachedRoute = '';
        let cachedPath = '';
        this.$root.getRootRoute = () => {
            if (this.$route.path != cachedPath) {
                cachedPath = this.$route.path;
                const m = cachedPath.match(/^(\/[^/]*).*$/i);
                cachedRoute = (m ? m[1] : this.$route.path);
            }
            return cachedRoute;
        }

        this.$root.isMobileDevice = /Android|webOS|iPhone|iPad|iPod|BlackBerry/i.test(navigator.userAgent);
        this.$root.setAppTitle = this.setAppTitle;

        //global keyHooks
        this.keyHooks = [];
        this.keyHook = (event) => {
            for (const hook of this.keyHooks)
                hook(event);
        }

        this.$root.addKeyHook = (hook) => {
            if (this.keyHooks.indexOf(hook) < 0)
                this.keyHooks.push(hook);
        }

        this.$root.removeKeyHook = (hook) => {
            const i = this.keyHooks.indexOf(hook);
            if (i >= 0)
                this.keyHooks.splice(i, 1);
        }

        document.addEventListener('keyup', (event) => {
            this.keyHook(event);
        });
        document.addEventListener('keypress', (event) => {
            this.keyHook(event);
        });
        document.addEventListener('keydown', (event) => {
            this.keyHook(event);
        });        
    }

    mounted() {
        this.$root.api = this.$refs.api;
        this.$root.notify = this.$refs.notify;
        this.$root.stdDialog = this.$refs.stdDialog;

        this.setAppTitle();
    }

    applyColorScheme(scheme) {
        if (this._mediaQuery) {
            this._mediaQuery.removeEventListener('change', this._mediaQueryHandler);
            this._mediaQuery = null;
            this._mediaQueryHandler = null;
        }

        if (scheme === 'light') {
            this.$q.dark.set(false);
        } else if (scheme === 'dark') {
            this.$q.dark.set(true);
        } else {
            // 'system'
            this.$q.dark.set('auto');
            if (typeof window !== 'undefined') {
                this._mediaQuery = window.matchMedia('(prefers-color-scheme: dark)');
                this._mediaQueryHandler = (e) => { this.$q.dark.set(e.matches); };
                this._mediaQuery.addEventListener('change', this._mediaQueryHandler);
            }
        }
    }

    get settings() {
        return this.$store.state.settings;
    }

    get config() {
        return this.$store.state.config;
    }

    get rootRoute() {
        return this.$root.getRootRoute();
    }

    setAppTitle(title) {
        if (title) {
            document.title = title;
        }
    }
}

export default vueComponent(App);
//-----------------------------------------------------------------------------
</script>

<style scoped>
</style>

<style>
:root {
    --toolbar-bg: #e0f7fa;
    --toolbar-btn-bg: #fff9c4;
    --link-color: #1565c0;
    --text-secondary: #555555;
    --border-color: #bbbbbb;
    --separator-color: #dddddd;
    --row-odd-bg: #e8e8e8;
}

.body--dark {
    --toolbar-bg: #1a2332;
    --toolbar-btn-bg: #2d3748;
    --link-color: #64b5f6;
    --text-secondary: #aaaaaa;
    --border-color: #444444;
    --separator-color: #333333;
    --row-odd-bg: #2a2a2a;
}

body, html, #app {
    margin: 0;
    padding: 0;
    width: 100%;
    height: 100%;
    font: normal 13px Web Default;
}

.dborder {
    border: 2px solid yellow;
}

.icon-rotate {
    vertical-align: middle;
    animation: rotating 2s linear infinite;
}

.q-dialog__inner--minimized {
    padding: 10px !important;
}

.q-dialog__inner--minimized > div {
    max-height: 100% !important;
    max-width: 800px !important;
}

@keyframes rotating { 
    from { 
        transform: rotate(0deg); 
    } to { 
        transform: rotate(360deg); 
    }
}

@font-face {
    font-family: 'Web Default';
    src: url('fonts/web-default.ttf') format('truetype');
}

@font-face {
    font-family: 'Verdana';
    font-weight: bold;
    src: url('fonts/web-default-bold.ttf') format('truetype');
}
</style>
