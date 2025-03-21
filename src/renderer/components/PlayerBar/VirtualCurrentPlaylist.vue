<template>
    <div class="current-list current-list--virtual">
        <CenteredTip v-if="queueEmpty"
            tip="列表里什么都没有，快去找几首歌吧 φ(≧ω≦*)♪"
            icon="music_note"></CenteredTip>
        <template v-else>
            <div class="actions">
                <span class="count">{{titleText}}</span>
                <div v-show="showFindInput">
                    <mu-text-field ref="findInput"
                        v-model="findInput"
                        action-icon="close"
                        :action-click="toggleFindInput"></mu-text-field>
                </div>
                <div class="buttons"
                    v-show="!showFindInput">
                    <mu-button flat
                        small
                        @click="toggleFindInput">
                        <mu-icon left
                            value="find_in_page"></mu-icon>
                        <span>查找</span>
                    </mu-button>
                    <mu-button flat
                        small
                        @click="handleCollectAll">
                        <mu-icon left
                            value="library_add"></mu-icon>
                        <span>收藏</span>
                    </mu-button>
                    <mu-button flat
                        small
                        @click="handleClearPlaylist">
                        <mu-icon left
                            value="delete_sweep"></mu-icon>
                        <span>清空</span>
                    </mu-button>
                </div>
            </div>
            <RecycleScroller class="mu-list mu-list-dense"
                ref="scroller"
                :items="listToShow"
                :item-size="36"
                keyField="id"
                :buffer="180">
                <template v-slot="{item, index}">
                    <mu-ripple tag="a"
                        :id="`cur-list-${index}`"
                        class="mu-item-wrapper"
                        @click="handleListClick(index)">
                        <div class="mu-item">
                            <div class="mu-item-action">
                                <mu-icon v-if="item.id == playing.id"
                                    color="secondary"
                                    value="volume_up"
                                    :size="18"></mu-icon>
                                <span v-else>{{(indexMap.has(index) ? indexMap.get(index) : index) + 1}}</span>
                            </div>
                            <div class="mu-item-title">
                                {{item.name}}
                                <span class="track-artist mu-item-after-text"> - {{item.artistName}}</span>
                            </div>
                            <div class="mu-item-action current-list-after">
                                <mu-icon v-if="item.source"
                                    value="link"
                                    :size="22"
                                    :title="sourceTipText(item)"
                                    @click.stop="handleSourceClick(item.source)"></mu-icon>
                                <mu-icon value="close"
                                    :size="22"
                                    title="从列表中删除"
                                    @click.stop="handleRemove(index)"></mu-icon>
                            </div>
                        </div>
                    </mu-ripple>
                </template>
            </RecycleScroller>
        </template>
    </div>
</template>

<script>
import { mapActions } from 'vuex';

import { workerExecute } from '@/worker/message';
import CenteredTip from '@/components/CenteredTip.vue';

const SourceName = {
    list: '歌单',
    album: '专辑',
    artist: '歌手',
    search: '搜索',
    disk: '音乐云盘',
    radio: '私人 FM',
    djradio: '主播电台',
    recommend: '每日歌曲推荐'
};

const RouteName = {
    list: 'playlist',
    album: 'album',
    artist: 'artist',
    search: 'search',
    disk: 'disk',
    radio: 'radio',
    djradio: 'djradio',
    recommend: 'recommend'
};

export default {
    props: {
        shown: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            /** @type {Models.Track[]} */
            filteredList: [],
            showFindInput: false,
            savedScrollTop: 0,
            findInput: '',
            /** @type {Map<number, number>} */
            indexMap: new Map()
        };
    },
    computed: {
        /** @returns {import('@/store/modules/ui').State} */
        ui() { return this.$store.state.ui; },
        /** @returns {import('@/store/getters').PlayingGetter}*/
        playing() { return this.$store.getters.playing; },
        /** @returns {import('@/store/getters').QueueGetter}*/
        queue() { return this.$store.getters.queue; },
        /** @returns {boolean} */
        queueEmpty() {
            return this.queue.list.length === 0;
        },
        /** @returns {Models.Track[]} */
        listToShow() {
            if (this.showFindInput === true && this.findInput.length > 0) {
                return this.filteredList;
            }
            return this.queue.list;
        },
        /** @returns {string} */
        titleText() {
            if (this.ui.radioMode) return '私人 FM';
            if (this.showFindInput && this.findInput) return `找到 ${this.filteredList.length} 首`;
            return `共 ${this.queue.list.length} 首`;
        }
    },
    methods: {
        ...mapActions([
            'clearRadio',
            'removeRadio',
            'clearPlaylist',
            'playTrackIndex',
            'toggleCollectPopup',
            'removeTrackFromPlaylist'
        ]),
        toggleFindInput() {
            this.showFindInput = !this.showFindInput;
            if (this.showFindInput === true) {
                this.savedScrollTop = this.$refs.scroller.$el.scrollTop;
                const input = this.$refs.findInput.$el.getElementsByTagName('input')[0];
                if (input) {
                    this.$nextTick(() => input.focus());
                }
            } else {
                this.findInput = '';
                this.indexMap.clear();
                this.filteredList = [];
                const scrollTop = this.savedScrollTop;
                this.$nextTick(() => {
                    this.$refs.scroller.$el.scrollTo(0, scrollTop);
                });
                this.savedScrollTop = 0;
            }
        },
        handleCollectAll() {
            const ids = this.listToShow.map(t => t.id);
            this.toggleCollectPopup(ids);
        },
        async handleClearPlaylist() {
            if (this.ui.radioMode) {
                this.$confirm('真的要清空私人 FM 播放记录吗？', '提示').then(m => {
                    if (m.result === true) this.clearRadio();
                });
                return;
            }
            this.$confirm('真的要清空播放列表吗？', '提示').then(m => {
                if (m.result === true) this.clearPlaylist();
            });
        },
        /** @param {number} index */
        handleListClick(index) {
            let i = index;
            if (this.indexMap.size > 0 && this.indexMap.has(index)) {
                i = this.indexMap.get(index);
            }
            this.playTrackIndex(i);
        },
        /** @param {Models.Track} track */
        sourceTipText(track) {
            return `来自${SourceName[track.source.name]}`;
        },
        navigateToSource(location) {
            if (this.$route.name === 'player') {
                this.$router.replace(location);
            } else {
                this.$router.push(location).catch(() => { /* noop */ });
            }
        },
        handleSourceClick(source) {
            const name = RouteName[source.name];
            switch (source.name) {
                case 'list':
                case 'album':
                case 'artist':
                case 'djradio':
                    this.navigateToSource({ name, params: { id: source.id } });
                    break;
                case 'search':
                    this.navigateToSource({ name, query: { keyword: source.id, type: 'song' } });
                    break;
                case 'disk':
                case 'radio':
                case 'recommend':
                    this.navigateToSource({ name });
                    break;
                default:
                    break;
            }
            this.$emit('navigate');
        },
        /** @param {number} index */
        handleRemove(index) {
            let start = index;
            if (this.filteredList.length > 0) {
                start = this.indexMap.get(index);
                this.filteredList.splice(index, 1);
            }
            if (this.ui.radioMode) {
                this.removeRadio(this.queue.list[start]);
                return;
            }
            this.removeTrackFromPlaylist({ start, count: 1 });
        },
        /** @param {number} index */
        scrollTo(index) {
            if (this.queueEmpty) return;
            const top = this.$refs.scroller.$el.scrollTop;
            const offset = index * 36;
            if (top > offset || top + 324 < offset) {
                this.$refs.scroller.$el.scrollTo(0, offset - 144);
            }
        },
        updateFilteredList() {
            const kw = this.findInput;
            if (kw.length > 0) {
                workerExecute('filterTracks', kw, this.queue.list).then(res => {
                    this.filteredList = res.result;
                    this.indexMap = res.indexMap;
                });
            } else {
                this.filteredList = [];
                this.indexMap.clear();
            }
        }
    },
    watch: {
        shown(val) {
            if (val && !this.showFindInput) {
                this.$nextTick(() => {
                    this.scrollTo(this.queue.index);
                });
            }
        },
        findInput() {
            this.updateFilteredList();
        },
        ['queue.list']() {
            this.updateFilteredList();
        }
    },
    components: {
        CenteredTip
    }
};
</script>

<style lang="less">
.current-list {
    height: 100%;
    .actions {
        height: 36px;
        padding: 0 4px 0 12px;
        display: flex;
        align-items: center;
        opacity: 0.75;
        box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
        .count {
            margin-right: auto;
        }
        .mu-input {
            font-size: 14px;
            margin: 0;
            padding: 0;
            min-height: unset;
            line-height: 26px;
            .mu-text-field-input {
                height: unset;
            }
            .mu-input-action-icon {
                padding-right: 0;
                font-size: 20px;
            }
        }
        .mu-button-small {
            min-width: unset;
        }
    }
    .mu-list {
        padding: 0;
        height: calc(~'100% - 36px');
        .mu-item {
            padding: 0 4px;
            .mu-item-action {
                z-index: 1;
                min-width: 36px;
                justify-content: center;
            }
            .track-artist {
                font-size: 11px;
            }
            .current-list-after {
                opacity: 0;
                min-width: 50px;
                flex-direction: row;
                justify-content: flex-end;
            }
        }
        .vue-recycle-scroller__item-view.hover {
            background-color: rgba(0, 0, 0, 0.1);
            .current-list-after {
                opacity: 0.6;
            }
        }
    }
}
</style>
