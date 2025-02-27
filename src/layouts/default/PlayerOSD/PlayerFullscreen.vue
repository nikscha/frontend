<template>
  <v-dialog
    v-model="store.showFullscreenPlayer"
    fullscreen
    :scrim="false"
    transition="dialog-bottom-transition"
    z-index="9999"
  >
    <v-card :color="darkenBrightColors(coverImageColorCode, 70, 80)">
      <v-toolbar class="v-toolbar-default" color="transparent">
        <template #prepend>
          <Button icon @click="store.showFullscreenPlayer = false">
            <v-icon icon="mdi-chevron-down" />
          </Button>
        </template>
        <template #append>
          <Button icon @click="store.showFullscreenPlayer = false">
            <v-icon icon="mdi-dots-vertical" @click.stop="openQueueMenu" />
          </Button>
        </template>
      </v-toolbar>

      <!-- content -->
      <div class="main">
        <!-- left column: media thumb + details-->
        <div
          v-if="getBreakpointValue('bp7') || !store.showQueueItems"
          class="main-media-details"
        >
          <div
            v-if="$vuetify.display.height > 600"
            class="main-media-details-image"
          >
            <v-img
              v-if="
                !store.curQueueItem?.image &&
                store.activePlayer?.current_media?.image_url
              "
              style="max-width: 100%; width: auto; border-radius: 4px"
              :src="store.activePlayer.current_media.image_url"
            />
            <MediaItemThumb
              v-else
              :item="store.curQueueItem"
              :thumbnail="false"
              style="max-width: 100%; width: auto"
              :fallback="
                $vuetify.theme.current.dark ? imgCoverDark : imgCoverLight
              "
            />
          </div>
          <div class="main-media-details-track-info">
            <!-- player name as title if its powered off-->
            <v-card-title
              v-if="store.activePlayer?.powered == false"
              :style="`font-size: ${titleFontSize};`"
            >
              {{ store.activePlayer?.display_name }}
            </v-card-title>
            <!-- queue item media item + optional version-->
            <v-card-title
              v-else-if="store.curQueueItem?.media_item"
              :style="`font-size: ${titleFontSize};cursor:pointer;`"
              @click="itemClick(store.curQueueItem.media_item as MediaItemType)"
            >
              {{ store.curQueueItem.media_item.name }}
              <span
                v-if="
                  'version' in store.curQueueItem.media_item &&
                  store.curQueueItem.media_item.version
                "
                >({{ store.curQueueItem.media_item.version }})</span
              >
            </v-card-title>

            <!-- external source current media item present -->
            <v-card-title v-else-if="store.activePlayer?.current_media?.title">
              <div v-if="store.activePlayer?.current_media?.artist">
                {{ store.activePlayer?.current_media?.artist }}
              </div>
              <div
                v-if="store.activePlayer?.current_media?.title"
                :style="`font-size: ${subTitleFontSize};`"
              >
                {{ store.activePlayer?.current_media?.title }}
              </div>
            </v-card-title>

            <!-- no player selected message -->
            <v-card-title
              v-else
              :style="`font-size: ${titleFontSize};cursor:pointer;`"
              @click="store.showPlayersMenu = true"
            >
              {{ store.activePlayer?.display_name || $t("no_player") }}
            </v-card-title>

            <!-- SUBTITLE -->

            <!-- SUBTITLE: player powered off -->
            <v-card-subtitle
              v-if="!store.activePlayer?.powered"
              class="text-h6 text-md-h5 text-lg-h4"
            >
              {{ $t("off") }}
            </v-card-subtitle>

            <!-- subtitle: radio station stream title -->
            <v-card-subtitle
              v-if="
                store.curQueueItem?.media_item?.media_type == MediaType.RADIO &&
                store.curQueueItem?.streamdetails?.stream_title
              "
              class="text-h6 text-md-h5 text-lg-h4"
              @click="
                radioTitleClick(store.curQueueItem.streamdetails.stream_title)
              "
            >
              {{ store.curQueueItem.streamdetails.stream_title }}
            </v-card-subtitle>

            <!-- subtitle: album -->
            <v-card-subtitle
              v-if="
                store.curQueueItem?.media_item &&
                'album' in store.curQueueItem.media_item &&
                store.curQueueItem.media_item.album
              "
              :style="`font-size: ${subTitleFontSize};cursor:pointer;`"
              @click="itemClick(store.curQueueItem.media_item.album as Album)"
            >
              {{ store.curQueueItem.media_item.album.name }}
            </v-card-subtitle>

            <!-- subtitle: artist(s) -->
            <!-- TODO enumerate all artists -->
            <v-card-subtitle
              v-if="
                store.curQueueItem?.media_item &&
                'artists' in store.curQueueItem.media_item &&
                store.curQueueItem.media_item.artists.length
              "
              :style="`font-size: ${subTitleFontSize};cursor:pointer;`"
              @click="
                itemClick(store.curQueueItem.media_item.artists[0] as Artist)
              "
            >
              {{ store.curQueueItem.media_item.artists[0].name }}
            </v-card-subtitle>

            <!-- subtitle: other source active -->
            <v-card-subtitle
              v-else-if="
                store.activePlayer?.active_source !=
                store.activePlayer?.player_id
              "
            >
              {{
                $t("external_source_active", [
                  store.activePlayer?.active_source,
                ])
              }}
            </v-card-subtitle>

            <!-- subtitle: queue empty -->
            <v-card-subtitle
              v-else-if="
                store.activePlayerQueue && store.activePlayerQueue.items == 0
              "
              :style="`font-size: ${subTitleFontSize}`"
            >
              {{ $t("queue_empty") }}
            </v-card-subtitle>

            <!-- streamdetails/contenttype button-->
            <div style="margin: auto; padding-top: 20px">
              <QualityDetailsBtn />
            </div>
          </div>
        </div>

        <!-- right column: queue items-->
        <div v-if="store.showQueueItems" class="main-queue-items">
          <v-tabs
            v-model="activeQueuePanel"
            hide-slider
            density="compact"
            @click="activeQueuePanelClick"
          >
            <v-tab :value="0">
              {{ $t("queue") }}
              <v-badge
                color="grey"
                :content="
                  (store.activePlayerQueue?.items || 0) -
                  (store.activePlayerQueue?.current_index || 0)
                "
                inline
              />
            </v-tab>
            <v-tab :value="1">
              {{ $t("played") }}
              <v-badge
                color="grey"
                :content="store.activePlayerQueue?.current_index"
                inline
              />
            </v-tab>
          </v-tabs>
          <div class="queue-items-scroll-box">
            <v-infinite-scroll
              v-if="!tempHide"
              :onLoad="loadNextPage"
              :empty-text="''"
              height="100%"
            >
              <!-- list view -->
              <v-virtual-scroll
                :item-height="70"
                max-height="90%"
                :items="activeQueuePanel == 0 ? nextItems : previousItems"
              >
                <template #default="{ item }">
                  <ListItem
                    link
                    :show-menu-btn="true"
                    @click.stop="(e) => openQueueItemMenu(e, item)"
                    @menu.stop="(e) => openQueueItemMenu(e, item)"
                  >
                    <template #prepend>
                      <div class="media-thumb listitem-media-thumb">
                        <MediaItemThumb size="50" :item="item" />
                      </div>
                    </template>
                    <template #title>
                      {{ item.name }}
                    </template>
                    <template #subtitle>
                      {{ formatDuration(item.duration) }}
                      <span
                        v-if="
                          item.media_item &&
                          'album' in item.media_item &&
                          item.media_item.album
                        "
                      >
                        | {{ item.media_item.album.name }}</span
                      >
                    </template>
                  </ListItem>
                </template>
              </v-virtual-scroll>
            </v-infinite-scroll>
          </div>
        </div>

        <!-- right column: media image (on small but wide screens)-->
        <div
          v-if="!store.showQueueItems && $vuetify.display.height <= 600"
          class="main-queue-items"
        >
          <div class="main-media-details-image main-media-details-image-alt">
            <MediaItemThumb
              :item="store.curQueueItem"
              :thumbnail="false"
              style="max-width: 100%; width: auto"
            />
          </div>
        </div>
      </div>

      <!-- player controls (always at bottom)-->
      <div class="player-bottom">
        <!-- timeline / progressbar-->
        <div class="row" style="margin-left: 5%; margin-right: 5%">
          <PlayerTimeline :show-labels="true" />
        </div>

        <!-- main media control buttons (play, next, previous etc.)-->
        <div class="media-controls">
          <ResponsiveIcon
            v-if="store.activePlayerQueue"
            :disabled="!store.curQueueItem?.media_item"
            :icon="
              store.curQueueItem?.media_item?.favorite
                ? 'mdi-heart'
                : 'mdi-heart-outline'
            "
            :title="$t('tooltip.favorite')"
            :type="'btn'"
            class="media-controls-item"
            max-height="30px"
            @click="onHeartBtnClick"
          />
          <ShuffleBtn
            v-if="$vuetify.display.mdAndUp"
            :player-queue="store.activePlayerQueue"
            class="media-controls-item"
            max-height="30px"
          />
          <PreviousBtn
            :player="store.activePlayer"
            :player-queue="store.activePlayerQueue"
            class="media-controls-item"
            max-height="45px"
          />
          <PlayBtn
            :player="store.activePlayer"
            :player-queue="store.activePlayerQueue"
            class="media-controls-item"
            max-height="100px"
          />
          <NextBtn
            :player="store.activePlayer"
            :player-queue="store.activePlayerQueue"
            class="media-controls-item"
            max-height="45px"
          />
          <RepeatBtn
            v-if="$vuetify.display.mdAndUp"
            :player-queue="store.activePlayerQueue"
            class="media-controls-item"
            max-height="35px"
          />
          <QueueBtn
            v-if="store.activePlayerQueue"
            class="media-controls-item"
            max-height="30px"
          />
        </div>

        <!-- volume control -->
        <div
          v-if="store.activePlayer"
          class="row"
          style="margin-left: 5%; margin-right: 5%"
        >
          <PlayerVolume
            width="100%"
            :is-powered="store.activePlayer?.powered"
            :disabled="!store.activePlayer || !store.activePlayer?.powered"
            :model-value="Math.round(store.activePlayer?.group_volume || 0)"
            prepend-icon="mdi-volume-minus"
            append-icon="mdi-volume-plus"
            @update:model-value="
              store.activePlayer!.group_childs.length > 0
                ? api.playerCommandGroupVolume(store.activePlayerId!, $event)
                : api.playerCommandVolumeSet(store.activePlayerId!, $event)
            "
            @click:prepend="
              store.activePlayer!.group_childs.length > 0
                ? api.playerCommandGroupVolume(
                    store.activePlayerId!,
                    store.activePlayer!.group_volume - 5,
                  )
                : api.playerCommandVolumeSet(
                    store.activePlayerId!,
                    store.activePlayer!.volume_level - 5,
                  )
            "
            @click:append="
              store.activePlayer!.group_childs.length > 0
                ? api.playerCommandGroupVolume(
                    store.activePlayerId!,
                    store.activePlayer!.group_volume + 5,
                  )
                : api.playerCommandVolumeSet(
                    store.activePlayerId!,
                    store.activePlayer!.volume_level + 5,
                  )
            "
          />
        </div>

        <!-- player select button -->
        <div
          v-if="$vuetify.display.height > 800"
          class="row"
          style="
            height: 70px;
            display: flex;
            justify-content: center;
            align-items: center;
            padding-bottom: 15px;
            padding-top: 15px;
          "
        >
          <v-btn
            class="responsive-icon-holder-btn"
            variant="outlined"
            @click="store.showPlayersMenu = true"
          >
            <v-icon :icon="store.activePlayer?.icon || 'mdi-speaker'" />
            {{ store.activePlayer ? getPlayerName(store.activePlayer) : "" }}
          </v-btn>
        </div>
      </div>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import { ref, computed, watch, onMounted, onBeforeUnmount } from "vue";
import MediaItemThumb from "@/components/MediaItemThumb.vue";
import api from "@/plugins/api";
import {
  Album,
  Artist,
  EventMessage,
  EventType,
  MediaItemType,
  MediaType,
  QueueItem,
  Track,
} from "@/plugins/api/interfaces";
import { store } from "@/plugins/store";
import PlayerTimeline from "./PlayerTimeline.vue";
import { getBreakpointValue } from "@/plugins/breakpoint";
import Button from "@/components/mods/Button.vue";
import ResponsiveIcon from "@/components/mods/ResponsiveIcon.vue";
import ListItem from "@/components/mods/ListItem.vue";
import vuetify from "@/plugins/vuetify";
import PlayBtn from "@/layouts/default/PlayerOSD/PlayerControlBtn/PlayBtn.vue";
import NextBtn from "@/layouts/default/PlayerOSD/PlayerControlBtn/NextBtn.vue";
import PreviousBtn from "@/layouts/default/PlayerOSD/PlayerControlBtn/PreviousBtn.vue";
import ShuffleBtn from "@/layouts/default/PlayerOSD/PlayerControlBtn/ShuffleBtn.vue";
import RepeatBtn from "@/layouts/default/PlayerOSD/PlayerControlBtn/RepeatBtn.vue";
import PlayerVolume from "@/layouts/default/PlayerOSD/PlayerVolume.vue";
import QueueBtn from "./PlayerControlBtn/QueueBtn.vue";
import QualityDetailsBtn from "@/components/QualityDetailsBtn.vue";
import {
  imgCoverLight,
  imgCoverDark,
} from "@/components/QualityDetailsBtn.vue";
import router from "@/plugins/router";
import {
  ImageColorPalette,
  darkenBrightColors,
  formatDuration,
  getPlayerName,
  sleep,
} from "@/helpers/utils";
import { eventbus } from "@/plugins/eventbus";
import { useDisplay } from "vuetify";
import { useI18n } from "vue-i18n";
import { ContextMenuItem } from "../ItemContextMenu.vue";
import { getPlayerMenuItems } from "@/helpers/player_menu_items";

const { t } = useI18n();
const { name } = useDisplay();

interface Props {
  colorPalette: ImageColorPalette;
}
const props = defineProps<Props>();

// Local refs
const queueItems = ref<QueueItem[]>([]);
const coverImageColorCode = ref<string>("");
const activeQueuePanel = ref(0);
const tempHide = ref(false);

// Computed properties

const nextItems = computed(() => {
  if (store.activePlayerQueue) {
    return queueItems.value.slice(store.activePlayerQueue.current_index);
  } else return [];
});
const previousItems = computed(() => {
  if (store.activePlayerQueue) {
    return queueItems.value.slice(0, store.activePlayerQueue.current_index);
  } else return [];
});

const titleFontSize = computed(() => {
  switch (name.value) {
    case "xs":
      return "1.2em";
    case "sm":
      return "1.6em";
    case "md":
      return "2em";
    case "lg":
      return store.showQueueItems ? "1.5em" : "2.5em";
    case "xl":
      return store.showQueueItems ? "1.6em" : "3em";
    case "xxl":
      return store.showQueueItems ? "1.7em" : "3.2em";
    default:
      return "1.0em.";
  }
});

const subTitleFontSize = computed(() => {
  switch (name.value) {
    case "xs":
      return "1.0em";
    case "sm":
      return "1.2em";
    case "md":
      return "1.7em";
    case "lg":
      return store.showQueueItems ? "1.0em" : "1.6em";
    case "xl":
      return store.showQueueItems ? "1.2em" : "2em";
    case "xxl":
      return store.showQueueItems ? "1.2em" : "2em";
    default:
      return "1.0em.";
  }
});

// methods

const itemClick = function (item: MediaItemType) {
  // mediaItem in the list is clicked
  store.showFullscreenPlayer = false;
  router.push({
    name: item.media_type,
    params: {
      itemId: item.item_id,
      provider: item.provider,
    },
  });
};

const radioTitleClick = function (streamTitle: string) {
  // radio station title clicked
  store.globalSearchTerm = streamTitle;
  router.push({ name: "search" });
  store.showFullscreenPlayer = false;
};

const openQueueItemMenu = function (evt: Event, item: QueueItem) {
  const itemIndex = queueItems.value.indexOf(item);
  const menuItems = [
    {
      label: "play_now",
      labelArgs: [],
      action: () => {
        queueCommand(item, "play_now");
      },
      icon: "mdi-play-circle-outline",
      disabled:
        itemIndex === store.activePlayerQueue?.current_index ||
        itemIndex === store.activePlayerQueue?.index_in_buffer,
    },
    {
      label: "play_next",
      labelArgs: [],
      action: () => {
        queueCommand(item, "move_next");
      },
      icon: "mdi-skip-next-circle-outline",
      disabled: itemIndex <= (store.activePlayerQueue?.index_in_buffer || 0),
    },
    {
      label: "queue_move_up",
      labelArgs: [],
      action: () => {
        queueCommand(item, "up");
      },
      icon: "mdi-arrow-up",
      disabled: itemIndex <= (store.activePlayerQueue?.index_in_buffer || 0),
    },
    {
      label: "queue_move_down",
      labelArgs: [],
      action: () => {
        queueCommand(item, "down");
      },
      icon: "mdi-arrow-down",
      disabled: itemIndex <= (store.activePlayerQueue?.index_in_buffer || 0),
    },
    {
      label: "queue_delete",
      labelArgs: [],
      action: () => {
        queueCommand(item, "delete");
      },
      icon: "mdi-delete",
      disabled: itemIndex <= (store.activePlayerQueue?.index_in_buffer || 0),
    },
  ];
  if (item?.media_item?.media_type == MediaType.TRACK) {
    menuItems.push({
      label: "show_info",
      labelArgs: [],
      action: () => {
        itemClick(item.media_item as Track);
      },
      icon: "mdi-information-outline",
      disabled: false,
    });
  }
  eventbus.emit("contextmenu", {
    items: menuItems,
    posX: (evt as PointerEvent).clientX,
    posY: (evt as PointerEvent).clientY,
  });
};

const openQueueMenu = function (evt: Event) {
  if (!store.activePlayer) return;
  eventbus.emit("contextmenu", {
    items: getPlayerMenuItems(store.activePlayer, store.activePlayerQueue),
    posX: (evt as PointerEvent).clientX,
    posY: (evt as PointerEvent).clientY,
  });
};

const queueCommand = function (item: QueueItem | undefined, command: string) {
  if (!item || !store.activePlayerQueue) return;
  if (command == "play_now") {
    api.queueCommandPlayIndex(
      store.activePlayerQueue?.queue_id,
      item.queue_item_id,
    );
  } else if (command == "move_next") {
    api.queueCommandMoveNext(
      store.activePlayerQueue?.queue_id,
      item.queue_item_id,
    );
  } else if (command == "up") {
    api.queueCommandMoveUp(
      store.activePlayerQueue?.queue_id,
      item.queue_item_id,
    );
  } else if (command == "down") {
    api.queueCommandMoveDown(
      store.activePlayerQueue?.queue_id,
      item.queue_item_id,
    );
  } else if (command == "delete") {
    api.queueCommandDelete(
      store.activePlayerQueue?.queue_id,
      item.queue_item_id,
    );
  }
};

const loadItems = async function (clear = false) {
  tempHide.value = true;
  if (clear) {
    queueItems.value = [];
  }

  if (store.activePlayerQueue) {
    const offset = queueItems.value.length;
    const limit = (store.activePlayerQueue.current_index || 0) + 50;
    const result = await api.getPlayerQueueItems(
      store.activePlayerQueue.queue_id,
      limit,
      offset,
    );
    queueItems.value.push(...result);
  }
  tempHide.value = false;
};

// eslint-disable-next-line @typescript-eslint/no-explicit-any
const loadNextPage = function ({ done }: { done: any }) {
  if (!store.activePlayerQueue || store.activePlayerQueue.items == 0) {
    done("empty");
    return;
  }
  if (queueItems.value.length >= store.activePlayerQueue?.items) {
    done("empty");
    return;
  }

  loadItems(false).then(() => {
    done("ok");
  });
};

// listen for item updates to refresh items when that happens
onMounted(() => {
  const unsub = api.subscribe(
    EventType.QUEUE_ITEMS_UPDATED,
    (evt: EventMessage) => {
      if (evt.object_id != store.activePlayerQueue?.queue_id) return;
      loadItems(true);
    },
  );
  onBeforeUnmount(unsub);
});

const onHeartBtnClick = function (evt: PointerEvent | MouseEvent) {
  // the heart icon/button was clicked
  if (!store.curQueueItem?.media_item) return;
  if (!store.curQueueItem.media_item.favorite) {
    api.toggleFavorite(store.curQueueItem.media_item);
    return;
  }

  const menuItems: ContextMenuItem[] = [
    {
      label: "favorites_remove",
      labelArgs: [],
      action: () => {
        api.toggleFavorite(store.curQueueItem!.media_item as MediaItemType);
      },
      icon: "mdi-heart",
    },
    {
      label: "add_playlist",
      labelArgs: [],
      action: () => {
        eventbus.emit("playlistdialog", {
          items: [store.curQueueItem!.media_item as MediaItemType],
        });
      },
      icon: "mdi-plus-circle-outline",
    },
  ];

  // open the contextmenu by emitting the event
  eventbus.emit("contextmenu", {
    items: menuItems,
    posX: evt.clientX,
    posY: evt.clientY,
  });
};

const activeQueuePanelClick = function () {
  // this hack is needed in order to force refresh the infinite scroller
  // otherwise it will not attempt to load more items if the end was reached
  // and then we load new items with a different size
  tempHide.value = true;
  sleep(100).then(() => {
    tempHide.value = false;
  });
};

// watchers
watch(
  () => store.activePlayerId,
  (val) => {
    loadItems(true);
  },
  { immediate: true },
);

watch(
  () => props.colorPalette,
  (result) => {
    if (!result.darkColor || !result.lightColor) {
      coverImageColorCode.value = vuetify.theme.current.value.dark
        ? "#000"
        : "#fff";
    } else {
      coverImageColorCode.value = vuetify.theme.current.value.dark
        ? result.darkColor
        : result.lightColor;
    }
  },
);
</script>

<style scoped>
.main {
  display: flex;
  min-height: 50% !important;
  height: 50% !important;
  max-height: 65% !important;
  padding-bottom: 5px;
}

.main-media-details {
  flex: 50%;
  max-width: 100%;
  width: 50%;
}

.main-queue-items {
  flex: 50%;
  height: 100%;
  max-width: 100%;
  padding-right: 10px;
  padding-left: 15px;
}

.queue-items-scroll-box {
  max-height: 100%;
  overflow-y: scroll;
}

.queue-items-scroll-box::-webkit-scrollbar {
  display: none; /* Safari and Chrome */
}

.v-infinite-scroll--vertical {
  overflow-y: unset;
}

.main-media-details-image {
  min-height: 50%;
  max-height: 80%;
  height: 60%;
  align-content: center;
  padding-left: 20px;
  padding-right: 20px;
}
.main-media-details-image.v-img {
  width: auto;
}

.main-media-details-image-alt {
  height: 100% !important;
  max-height: 100% !important;
  align-content: center;
  padding: 0px !important;
}

.main-media-details-track-info {
  height: 20%;
  max-height: 20%;
  align-content: center;
  text-align: center;
  padding: 20px;
}

.player-bottom {
  max-height: 35% !important;
  min-height: 25% !important;
  height: 30% !important;
  margin-top: auto;
  bottom: 0;
  position: unset !important;
  padding-bottom: 5%;
}

.track-info {
  margin-top: 10px;
  margin-bottom: 10px;
  padding-top: 2vh;
  padding-bottom: 2vh;
  text-align: center;
  line-height: 10%;
}

.track-info-subtitle {
  opacity: 0.5;
}

.v-tab {
  opacity: 0.5;
}
.v-tab-item--selected {
  opacity: 1;
}

.media-controls {
  display: flex;
  flex: 1 1 auto;
  align-items: center;
  justify-content: center;
  max-width: 100%;
  padding: 15px;
  height: 100px;
}

.media-controls-item {
  margin: 0 10px;
  width: 100%;
  height: 100%;
}

@media (max-width: 768px) {
  .media-controls-item {
    margin: 0 5px;
    width: 100%;
    height: 100%;
  }
}

.row {
  align-content: center;
}

.row-centered {
  justify-content: center;
  max-width: 100%;
  padding: 15px;
}

.media-controls > button {
  flex: 1 1 auto;
}

.media-controls-bottom {
  display: flex;
  flex: 0 1 auto;
  justify-content: center;
  align-items: center;
}

.media-controls-bottom > div {
  flex-basis: 0;
  flex-grow: 1;
  max-width: 100%;
}

.media-controls > div {
  width: calc(100% / 3);
}

.mediacontrols-right {
  display: table-cell;
  text-align: right;
  vertical-align: middle;
}

.mediacontrols-right > div {
  display: inline-flex;
  align-items: center;
}

.v-toolbar :deep(.v-toolbar-title) {
  text-align: center;
}

.responsive-icon-holder-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0.62;
  cursor: pointer !important;
}

.responsive-icon-holder-btn:focus,
.responsive-icon-holder-btn:hover {
  opacity: 1;
}
</style>
