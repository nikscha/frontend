<template>
  <!-- now playing media -->
  <v-list-item
    style="height: auto; width: fit-content; margin: 0px; padding: 0px"
    lines="two"
  >
    <template #prepend>
      <div
        class="media-thumb player-media-thumb"
        :style="`cursor: pointer;height: ${
          getBreakpointValue({ breakpoint: 'phone' }) ? 50 : 64
        }px; width: ${
          getBreakpointValue({ breakpoint: 'phone' }) ? 50 : 64
        }px; `"
        @click="store.showFullscreenPlayer = true"
      >
        <MediaItemThumb
          v-if="store.curQueueItem?.media_item || store.curQueueItem?.image"
          :item="store.curQueueItem?.media_item || store.curQueueItem"
          :fallback="imgCoverDark"
        />
        <div
          v-else-if="
            store.activePlayer?.powered &&
            store.activePlayer?.current_media?.image_url
          "
        >
          <v-img
            class="media-thumb"
            style="border-radius: 4px"
            size="55"
            :src="store.activePlayer.current_media.image_url"
          />
        </div>
        <div v-else class="icon-thumb">
          <v-icon
            size="35"
            :icon="
              store.activePlayer?.type == PlayerType.PLAYER &&
              store.activePlayer?.group_childs.length
                ? 'mdi-speaker-multiple'
                : store.activePlayer?.icon || 'mdi-speaker'
            "
            style="display: table-cell; opacity: 0.8"
          />
        </div>
      </div>
    </template>

    <!-- title -->
    <template #title>
      <div
        :style="{
          cursor: 'pointer',
          color: primaryColor,
        }"
      >
        <!-- player name as title if its powered off-->
        <div
          v-if="store.activePlayer?.powered == false"
          @click="store.showPlayersMenu = true"
        >
          {{ store.activePlayer?.display_name }}
        </div>
        <!-- queue item media item + optional version-->
        <div
          v-else-if="store.curQueueItem?.media_item"
          @click="store.showFullscreenPlayer = true"
        >
          {{ store.curQueueItem.media_item.name }}
          <span
            v-if="
              'version' in store.curQueueItem.media_item &&
              store.curQueueItem.media_item.version
            "
            >({{ store.curQueueItem.media_item.version }})</span
          >
        </div>
        <!-- queue item fallback: queue item name -->
        <div v-else-if="store.curQueueItem">
          {{ store.curQueueItem.name }}
        </div>
        <!-- external source current media item present -->
        <div v-else-if="store.activePlayer?.current_media?.title">
          {{ store.activePlayer.current_media.title }}
        </div>
        <!-- no player selected message -->
        <div
          v-else-if="!store.activePlayer"
          @click="store.showPlayersMenu = true"
        >
          {{ $t("no_player") }}
        </div>
        <!-- queue empty message -->
        <div
          v-else-if="
            store.activePlayerQueue && store.activePlayerQueue.items == 0
          "
          class="line-clamp-1"
        >
          {{ $t("queue_empty") }}
        </div>
      </div>
    </template>
    <!-- append chip(s): quality -->
    <template #append>
      <!-- format -->
      <v-chip
        v-if="
          streamDetails?.audio_format.content_type &&
          !getBreakpointValue({ breakpoint: 'phone' }) &&
          showQualityDetailsBtn
        "
        :disabled="
          !store.activePlayerQueue ||
          !store.activePlayerQueue?.active ||
          store.activePlayerQueue?.items == 0
        "
        class="player-track-content-type"
        :style="
          $vuetify.theme.current.dark
            ? 'color: #000; background: #fff; margin-left: 15px;'
            : 'color: #fff; background: #000; margin-left: 15px;'
        "
        label
        :ripple="false"
        v-bind="props"
      >
        <div class="d-flex justify-center" style="width: 100%">
          {{ streamDetails.audio_format.content_type.toUpperCase() }}
        </div>
      </v-chip>
    </template>
    <!-- subtitle -->
    <template #subtitle>
      <div
        :style="{
          cursor: 'pointer',
          color: primaryColor,
        }"
        class="line-clamp-1"
        @click="store.showFullscreenPlayer = true"
      >
        <!-- player powered off -->
        <div v-if="!store.activePlayer?.powered">
          {{ $t("off") }}
        </div>
        <!-- track: artists(s) + album -->
        <div
          v-else-if="
            store.curQueueItem &&
            store.curQueueItem.media_item?.media_type == MediaType.TRACK &&
            'album' in store.curQueueItem.media_item &&
            store.curQueueItem.media_item.album &&
            !props.showOnlyArtist
          "
        >
          {{ getArtistsString(store.curQueueItem.media_item.artists) }} •
          {{ store.curQueueItem.media_item.album.name }}
        </div>
        <!-- track fallback: (only artist, no album) -->
        <div
          v-else-if="
            store.curQueueItem &&
            store.curQueueItem.media_item &&
            'artists' in store.curQueueItem.media_item &&
            store.curQueueItem.media_item.artists.length > 0
          "
        >
          {{ store.curQueueItem.media_item.artists[0].name }}
        </div>
        <!-- radio live metadata -->
        <div
          v-else-if="store.curQueueItem?.streamdetails?.stream_title"
          class="line-clamp-1"
        >
          {{ store.curQueueItem?.streamdetails?.stream_title }}
        </div>
        <!-- other description -->
        <div
          v-else-if="
            store.curQueueItem &&
            store.curQueueItem.media_item?.metadata.description
          "
          class="line-clamp-1"
        >
          {{ store.curQueueItem.media_item.metadata.description }}
        </div>
        <!-- external source artist -->
        <div v-else-if="store.activePlayer?.current_media?.artist">
          {{ store.activePlayer.current_media.artist }}
        </div>
        <!-- 3rd party source active -->
        <div
          v-else-if="
            store.activePlayer?.active_source != store.activePlayer?.player_id
          "
          class="line-clamp-1"
        >
          {{
            $t("external_source_active", [store.activePlayer?.active_source])
          }}
        </div>
        <!-- active player -->
        <div v-if="store.activePlayer && store.activePlayer?.powered">
          {{ getPlayerName(store.activePlayer) }}
        </div>
      </div>
    </template>
  </v-list-item>
  <PlayerFullscreen
    :show-fullscreen="store.showFullscreenPlayer"
    :color-palette="colorPalette"
  />
</template>

<script setup lang="ts">
import { computed } from "vue";

import { MediaType, PlayerType } from "@/plugins/api/interfaces";
import { store } from "@/plugins/store";
import MediaItemThumb from "@/components/MediaItemThumb.vue";
import {
  ImageColorPalette,
  getArtistsString,
  getPlayerName,
} from "@/helpers/utils";
import PlayerFullscreen from "./PlayerFullscreen.vue";
import { imgCoverDark } from "@/components/QualityDetailsBtn.vue";
import { getBreakpointValue } from "@/plugins/breakpoint";

// properties
interface Props {
  showOnlyArtist?: boolean;
  showQualityDetailsBtn?: boolean;
  colorPalette: ImageColorPalette;
  primaryColor: string;
}

const props = withDefaults(defineProps<Props>(), {
  showOnlyArtist: false,
  showQualityDetailsBtn: true,
  primaryColor: "",
});

// computed properties
const streamDetails = computed(() => {
  return store.activePlayerQueue?.current_item?.streamdetails;
});
</script>

<style>
.player-media-thumb {
  margin-right: 10px;
}

.player-track-content-type {
  height: 20px !important;
  padding: 5px !important;
  padding-right: 9px !important;
  padding-left: 9px !important;
  font-weight: 500;
  font-size: 10px !important;
  letter-spacing: 0.1em;
  border-radius: 2px;
  margin-right: 30px;
}

.icon-thumb {
  width: 55px;
  height: 55px;
  margin-top: 5px;
  border-radius: 4px;
  background-color: rgba(0, 0, 0, 0.3);
  display: inline-table;
}
</style>
