<template>
    <div class="player-bar">
        <audio id="playerbar-audio"
            ref="audio"
            :src="ui.audioSrc"></audio>
        <div class="cover"
            @click="handleCoverClick">
            <img :src="coverImgSrc"
                class="img"
                width="64"
                height="64">
            <mu-icon :value="coverIcon"
                :size="48"></mu-icon>
        </div>
        <div class="info">
            <div class="desc">
                <div class="name">
                    <span class="track">{{playing.name}}</span>
                    <span class="artist">{{playing.artistName}}</span>
                </div>
                <div class="shortcut">
                    <div v-if="isDjRadioProgram">
                        <!-- hide like button for now -->
                        <div class="icon-placeholder"></div>
                    </div>
                    <div v-else title="喜欢">
                        <mu-checkbox uncheck-icon="favorite_border"
                            checked-icon="favorite"
                            color="red"
                            :inputValue="isFavorite"
                            @click="handleFavorite"></mu-checkbox>
                    </div>
                    <mu-menu :open.sync="volumeShown"
                        placement="top"
                        popover-class="playerbar-volume"
                        title="音量">
                        <mu-checkbox :uncheck-icon="iconVolume"
                            :checked-icon="iconVolume"
                            color="secondary"
                            :inputValue="volumeShown"
                            @wheel.passive="handleVolumeWheel"
                            @auxclick="handleVolumeMute"></mu-checkbox>
                        <template #content>
                            <div class="content"
                                @mouseenter="cancelHideVolume"
                                @mouseleave="scheduleHideVolume"
                                @wheel.passive="handleVolumeWheel">
                                <mu-slider :value="ui.audioVolume"
                                    :min="0"
                                    :max="100"
                                    :step="1"
                                    :disabled="ui.audioMute"
                                    :display-value="false"
                                    @change="handleVolumeChange"></mu-slider>
                                <span class="value">{{ui.audioVolume}}</span>
                            </div>
                        </template>
                    </mu-menu>
                    <div v-if="ui.radioMode"
                        title="不喜欢">
                        <mu-checkbox uncheck-icon="delete_outline"
                            @click="handleRadioDislike"></mu-checkbox>
                    </div>
                    <div v-else
                        title="循环模式">
                        <mu-checkbox :uncheck-icon="iconLoopMode"
                            @click="handleLoopMode"></mu-checkbox>
                    </div>
                    <div :title="ui.downloaded ? '已下载' : '下载'">
                        <mu-checkbox uncheck-icon="get_app"
                            checked-icon="done"
                            color="secondary"
                            :inputValue="ui.downloaded"
                            @click="handleDownload"></mu-checkbox>
                    </div>
                    <mu-menu :open.sync="currentListShown"
                        placement="top"
                        popover-class="playerbar-current-list"
                        title="播放列表">
                        <mu-checkbox uncheck-icon="queue_music"
                            checked-icon="queue_music"
                            color="secondary"
                            :inputValue="currentListShown"></mu-checkbox>
                        <template #content>
                            <CurrentPlaylist :shown="currentListShown"
                                @navigate="handleSourceNavigate"></CurrentPlaylist>
                        </template>
                    </mu-menu>
                </div>
            </div>
            <div class="progress">
                <mu-slider id="playerbar-progress"
                    :display-value="false"
                    :value="songProgress"
                    @change="handleProgressDrag"></mu-slider>
                <span class="time">{{ shortTime(timeCurrent) }} / {{ shortTime(timeTotal) }}</span>
            </div>
        </div>
        <div class="control">
            <mu-button fab
                small
                class="button"
                @click="handlePrev">
                <mu-icon value="skip_previous"></mu-icon>
            </mu-button>
            <mu-button fab
                class="button"
                @click="handlePlayPause">
                <mu-icon :value="iconPlayPause"></mu-icon>
            </mu-button>
            <mu-button fab
                small
                class="button"
                @click="handleNext">
                <mu-icon value="skip_next"></mu-icon>
            </mu-button>
        </div>
    </div>
</template>

<script>
import { mapActions } from 'vuex';

import Api from '@/api/ipc';
import CurrentPlaylist from './VirtualCurrentPlaylist.vue';
import {
    UPDATE_PLAYING_URL,
    SET_AUDIO_VOLUME,
    SET_AUDIO_PAUSED
} from '@/store/mutation-types';
import { LOOP_MODE } from '@/store/modules/playlist';
import { sizeImg, HiDpiPx } from '@/util/image';
import { shortTime } from '@/util/formatter';

import coverDefault from 'assets/img/cover_default.webp';

const VolumeIcon = [
    'volume_off',
    'volume_mute',
    'volume_down',
    'volume_up'
];

export default {
    data() {
        return {
            hasRetried: false,
            timeTotal: 0,
            timeCurrent: 0,
            shouldFavorite: null,
            volumeShown: false,
            volumeHideTimeoutId: -1,
            currentListShown: false
        };
    },
    methods: {
        shortTime,
        ...mapActions([
            'playAudio',
            'pauseAudio',
            'updateUiAudioSrc',
            'setAudioVolume',
            'playNextTrack',
            'playPreviousTrack',
            'updateFavoriteTrackIds',
            'favoriteTrack',
            'nextLoopMode',
            'likeRadio',
            'skipRadio',
            'trashRadio',
            'downloadTrack',
            'checkDownloaded'
        ]),
        handleCoverClick() {
            if (this.$route.name === 'player') {
                this.$router.back();
                return;
            }
            this.$router.push({ name: 'player' });
        },
        handlePrev() {
            this.playPreviousTrack();
        },
        handlePlayPause() {
            this.ui.audioSrc && (this.ui.paused ? this.playAudio() : this.pauseAudio());
        },
        getAudioCurrentTime() {
            return Math.trunc(this.$refs.audio.currentTime * 1000);
        },
        handleNext() {
            if (this.ui.radioMode === true) {
                this.skipRadio({
                    id: this.playing.id,
                    time: this.getAudioCurrentTime()
                });
            }
            this.playNextTrack();
        },
        handleProgressDrag(value) {
            this.$refs.audio.currentTime = (this.timeTotal / 1000) * (value / 100);
        },
        submitSongStartPlay() {
            if (this.user.loginValid) {
                Api.submitSongStartPlay(this.playing.id, this.user.info.id);
            }
        },
        submitSongPlayed() {
            if (this.user.loginValid) {
                Api.submitSongPlayed(this.playing.id, Math.round(this.timeTotal / 1000), this.playing.source, this.user.info.id);
            }
        },
        async handleFavorite() {
            if (!this.user.loginValid || !this.playing.id) {
                this.$toast.message('真的这么喜欢我吗 o(*////▽////*)q');
                return;
            }
            if (this.shouldFavorite !== null) return;
            this.shouldFavorite = !this.isFavorite;
            try {
                if (this.ui.radioMode) {
                    await this.likeRadio({
                        id: this.playing.id,
                        time: this.getAudioCurrentTime(),
                        like: this.shouldFavorite
                    });
                } else {
                    this.favoriteTrack({ id: this.playing.id, favorite: this.shouldFavorite })
                        .catch(e => this.$toast.message(e.msg));
                }
                // it would take some time for NetEase to update playlist cover
                // img, so we just wait 200 ms
                await new Promise(_ => setTimeout(() => _(), 200));
                await this.updateFavoriteTrackIds();
            } finally {
                this.shouldFavorite = null;
            }
        },
        cancelHideVolume() {
            if (this.volumeHideTimeoutId >= 0) {
                window.clearTimeout(this.volumeHideTimeoutId);
            }
        },
        scheduleHideVolume() {
            this.cancelHideVolume();
            this.volumeHideTimeoutId = setTimeout(() => {
                this.volumeHideTimeoutId = -1;
                this.volumeShown = false;
            }, 750);
        },
        handleVolumeChange(value) {
            let volume = value;
            if (value > 100) {
                volume = 100;
            } else if (volume < 0) {
                volume = 0;
            }
            if (volume === this.ui.audioVolume) return;
            let payload = { volume };
            const mute = volume === 0;
            if (this.ui.audioMute !== mute) {
                payload.mute = mute;
            }
            this.setAudioVolume(payload);
        },
        handleVolumeWheel(ev) {
            if (this.volumeShown === false) {
                this.volumeShown = true;
            }
            this.scheduleHideVolume();
            if (ev.deltaY > 0) {
                this.handleVolumeChange(this.ui.audioVolume - 5);
            } else {
                this.handleVolumeChange(this.ui.audioVolume + 5);
            }
        },
        handleVolumeMute() {
            this.setAudioVolume({ mute: !this.ui.audioMute });
        },
        handleLoopMode() {
            if (this.ui.radioMode) {
                this.$toast.message('私人 FM');
                return;
            }
            this.nextLoopMode();
            switch (this.queue.loopMode) {
                case LOOP_MODE.LIST:
                    this.$toast.message('列表顺序');
                    break;
                case LOOP_MODE.LOOP:
                    this.$toast.message('列表循环');
                    break;
                case LOOP_MODE.SINGLE:
                    this.$toast.message('单曲循环');
                    break;
                case LOOP_MODE.RANDOM:
                    this.$toast.message('随机播放');
                    break;
            }
        },
        handleRadioDislike() {
            if (!this.playing.id) {
                this.$toast.message('不跟你玩了，哼  o(￣ヘ￣o＃)');
                return;
            }
            const time = this.getAudioCurrentTime();
            this.trashRadio({ id: this.playing.id, time });
            this.playNextTrack();
        },
        handleSourceNavigate() {
            this.currentListShown = false;
        },
        async handleDownload() {
            if (this.ui.downloaded) {
                return;
            }
            if (this.ui.downloading) {
                this.$toast.message('已经在下载了呢  (*/ω＼*)');
                return;
            }
            if (!this.playing.id) {
                this.$toast.message('想下载什么呢  ヾ(´･ω･｀)ﾉ');
                return;
            }
            const result = await this.downloadTrack({ metadata: this.playing, quality: this.settings.bitRateDownload });
            if (!result.success) {
                this.$toast.error(result.error);
            }
        }
    },
    computed: {
        /** @returns {import('@/store/modules/ui').State}*/
        ui() { return this.$store.state.ui; },
        /** @returns {import('@/store/modules/user').State}*/
        user() { return this.$store.state.user; },
        /** @returns {import('@/store/modules/settings').State}*/
        settings() { return this.$store.state.settings; },
        /** @returns {import('@/store/getters').PlayingGetter}*/
        playing() { return this.$store.getters.playing; },
        /** @returns {import('@/store/getters').QueueGetter}*/
        queue() { return this.$store.getters.queue; },
        coverImgSrc() {
            if (this.ui.coverImgSrc) {
                return sizeImg(this.ui.coverImgSrc, HiDpiPx(64));
            }
            return coverDefault;
        },
        coverIcon() {
            if (this.$route.name === 'player') return 'fullscreen_exit';
            return 'fullscreen';
        },
        songProgress() {
            return 100 * this.timeCurrent / this.timeTotal || 0;
        },
        isDjRadioProgram() {
            const { source = {} } = this.playing;
            return (source && source.djradio);
        },
        isFavorite() {
            if (!this.user.loginValid || !this.playing) {
                return false;
            }
            if (this.shouldFavorite !== null) {
                return this.shouldFavorite;
            }
            return this.user.favTrackIds.includes(this.playing.id);
        },
        iconVolume() {
            if (this.ui.audioMute === true || this.ui.audioVolume <= 0) return VolumeIcon[0];
            if (this.ui.audioVolume <= 33) return VolumeIcon[1];
            if (this.ui.audioVolume <= 66) return VolumeIcon[2];
            return VolumeIcon[3];
        },
        iconLoopMode() {
            switch (this.queue.loopMode) {
                case LOOP_MODE.LIST:
                    return 'list';
                case LOOP_MODE.LOOP:
                    return 'repeat';
                case LOOP_MODE.SINGLE:
                    return 'repeat_one';
                case LOOP_MODE.RANDOM:
                    return 'shuffle';
            }
            return 'not_interested';
        },
        iconPlayPause() {
            return this.ui.paused ? 'play_arrow' : 'pause';
        }
    },
    mounted() {
        /** @type {HTMLAudioElement} */
        const _audioEl = this.$refs.audio;
        this.$root.$emit('audio-ready', _audioEl);

        if (this.ui.audioMute === true) {
            _audioEl.volume = 0;
        } else {
            _audioEl.volume = this.ui.audioVolume / 100;
        }

        const _updateTime = () => this.timeCurrent = _audioEl.currentTime * 1000;
        let _shouldUpdateTime = false;
        const _frame = () => {
            if (!_shouldUpdateTime) return;
            _updateTime();
            _scheduleFrame();
        };
        const _scheduleFrame = () => {
            const delay =  Math.ceil((1 - _audioEl.currentTime % 1) * 1000);
            setTimeout(() => requestAnimationFrame(_frame), delay);
            // this is not perfect. sometimes `_frame` happens few milliseconds
            // before next second arrives, although `delay` was right.
        };
        const _setUpdateTimeInterval = () => {
            if (_shouldUpdateTime) return;
            _shouldUpdateTime = true;
            _scheduleFrame();
        };
        const _unsetUpdateTimeInterval = () => _shouldUpdateTime = false;

        _audioEl.addEventListener('loadedmetadata', () => {
            _unsetUpdateTimeInterval();
            this.timeTotal = _audioEl.duration * 1000;
            this.timeCurrent = _audioEl.currentTime;
            if (this.ui.paused === false && _audioEl.paused) {
                _audioEl.play();
            }

            this.checkDownloaded({ metadata: this.playing });
        });

        _audioEl.addEventListener('seeking', _updateTime);

        _audioEl.addEventListener('playing', () => {
            _updateTime();
            _setUpdateTimeInterval();
        });

        _audioEl.addEventListener('pause', () => {
            _updateTime();
            _unsetUpdateTimeInterval();
        });

        _audioEl.addEventListener('ended', () => {
            _unsetUpdateTimeInterval();
            this.submitSongPlayed();
            if (this.queue.loopMode === LOOP_MODE.SINGLE) {
                _audioEl.play();
            } else {
                this.playNextTrack();
            }
        });

        _audioEl.addEventListener('error', () => {
            _unsetUpdateTimeInterval();
            if (!this.playing.id) return;
            if (!this.hasRetried) {
                this.hasRetried = true;
                this.updateUiAudioSrc({ ignoreCache: true });
                return;
            }
            this.hasRetried = false;
            this.$toast.message('这首歌听不了了，换下一首吧   (￣△￣;) ');
            setTimeout(i => {
                if (!this.ui.paused && i === this.playing.id) {
                    this.playNextTrack();
                }
            }, 3000, this.playing.id);
        });

        this.$store.subscribe(mutation => {
            switch (mutation.type) {
                case UPDATE_PLAYING_URL:
                    if (!mutation.payload) {
                        _unsetUpdateTimeInterval();
                        this.timeTotal = this.timeCurrent = 0;
                    }
                    break;
                case SET_AUDIO_VOLUME:
                    if (this.ui.audioMute === true) {
                        _audioEl.volume = 0;
                    } else {
                        _audioEl.volume = this.ui.audioVolume / 100;
                    }
                    break;
                case SET_AUDIO_PAUSED:
                    mutation.payload === true ? _audioEl.pause() : _audioEl.play();
                    break;
            }
        });
    },
    // TODO: unset `updateTimeInterval` created in `mounted()` hook
    // destroy `_audioEl` and so on ...
    // beforeDestroy() {},
    components: {
        CurrentPlaylist
    }
};
</script>

<style lang="less">
.player-bar {
    background-color: var(--background-color);
    box-shadow: 0 2px 12px rgba(0, 0, 0, 0.4);
    display: flex;
    user-select: none;
    cursor: default;
    height: 64px;
    position: fixed;
    width: 100%;
    bottom: 0;
    left: 0;
    z-index: 1;
    .cover {
        height: 64px;
        width: 64px;
        cursor: pointer;
        position: relative;
        .img,
        .mu-icon {
            transition: 0.3s opacity;
        }
        .mu-icon {
            top: 8px;
            left: 8px;
            opacity: 0;
            position: absolute;
        }
        &:hover {
            .img {
                opacity: 0.65;
            }
            .mu-icon {
                opacity: 1;
            }
        }
        &:active {
            .img {
                opacity: 0.3;
            }
        }
    }
    .info {
        font-size: 14px;
        padding: 0 12px;
        flex-grow: 1;
        overflow: hidden;
        .desc {
            margin-top: 8px;
            margin-bottom: 4px;
            display: flex;
            justify-content: space-between;
            .name {
                flex-grow: 1;
                display: flex;
                overflow: hidden;
                white-space: nowrap;
                line-height: 24px;
                .track,
                .artist {
                    overflow: hidden;
                    text-overflow: ellipsis;
                }
                .track {
                    min-width: min-content;
                    max-width: 400px;
                }
                .artist {
                    margin-left: 8px;
                    opacity: 0.6;
                }
            }
            .shortcut {
                flex-basis: calc(36px * 5);
                flex-shrink: 0;
                margin-left: 16px;
                display: flex;
                justify-content: space-between;
                font-size: 0;
                .icon-placeholder {
                    width: 24px;
                    height: 24px;
                }
            }
        }
        .progress {
            display: flex;
            .mu-slider {
                margin: 0;
            }
            .time {
                width: 110px;
                text-align: right;
            }
        }
    }
    .control {
        flex-basis: 160px;
        flex-shrink: 0;
        margin-right: 10px;
        display: flex;
        justify-content: space-between;
        align-items: center;
        .mu-fab-button {
            box-shadow: unset;
        }
    }
}

.playerbar-volume {
    border-radius: 0;
    height: 40px;
    width: 170px;
    top: unset !important;
    left: unset !important;
    right: 164px !important;
    bottom: 72px !important;
    .content {
        display: flex;
        align-items: center;
        justify-content: space-between;
        .mu-slider {
            margin: 8px 16px;
            width: 100px;
        }
        .value {
            margin-right: 8px;
        }
    }
}

.playerbar-current-list {
    border-radius: 0; // avoid 'repaint on scroll'
    width: 420px;
    height: 396px;
    font-size: 14px;
    position: fixed;
    top: unset !important;
    left: unset !important;
    right: 8px !important;
    bottom: 72px !important;
}
</style>
