<!--
  Global dialog to generate a Spotify-powered playlist via the recommendations API.
  Uses Vuetify components throughout to avoid focus-trap conflicts with reka-ui.
  Triggered via eventbus so it can be invoked from the Playlists toolbar.
-->
<template>
  <v-dialog v-model="showDialog" max-width="640" scrollable>
    <v-card>
      <v-card-title>{{ $t("spotify_recommendations_title") }}</v-card-title>
      <v-card-subtitle>
        {{ $t("spotify_recommendations_seeds_hint", [totalSeeds, 5]) }}
      </v-card-subtitle>

      <v-card-text>
        <!-- ── Seeds ── -->
        <v-tabs v-model="activeTab" density="compact" class="mb-2">
          <v-tab value="genres">
            {{ $t("genres") }}
            <span v-if="selectedGenres.length" class="ml-1 text-caption opacity-60">
              ({{ selectedGenres.length }})
            </span>
          </v-tab>
          <v-tab value="artists">
            {{ $t("artists") }}
            <span v-if="selectedArtists.length" class="ml-1 text-caption opacity-60">
              ({{ selectedArtists.length }})
            </span>
          </v-tab>
          <v-tab value="tracks">
            {{ $t("tracks") }}
            <span v-if="selectedTracks.length" class="ml-1 text-caption opacity-60">
              ({{ selectedTracks.length }})
            </span>
          </v-tab>
        </v-tabs>

        <v-window v-model="activeTab">
          <!-- Genre seeds -->
          <v-window-item value="genres">
            <v-text-field
              v-model="genreSearch"
              :placeholder="$t('search_filter_genres')"
              density="compact"
              variant="outlined"
              hide-details
              clearable
              class="mb-2"
            />
            <v-list
              density="compact"
              max-height="140"
              style="overflow-y: auto"
              class="border rounded"
            >
              <v-list-item
                v-for="genre in filteredGenres"
                :key="genre"
                :title="genre"
                :disabled="totalSeeds >= 5"
                @click="addGenre(genre)"
              />
              <v-list-item
                v-if="filteredGenres.length === 0"
                :title="$t('no_content')"
                disabled
              />
            </v-list>
          </v-window-item>

          <!-- Artist seeds -->
          <v-window-item value="artists">
            <v-text-field
              v-model="artistSearch"
              :placeholder="$t('search_artists')"
              density="compact"
              variant="outlined"
              hide-details
              clearable
              class="mb-2"
              @update:model-value="onArtistSearchInput"
            />
            <v-list
              density="compact"
              max-height="140"
              style="overflow-y: auto"
              class="border rounded"
            >
              <v-list-item v-if="artistSearchLoading" :title="$t('loading')" disabled />
              <v-list-item
                v-for="artist in artistResults"
                :key="artist.uri"
                :title="artist.name"
                :disabled="!canAddSeed(artist.uri, 'artist')"
                @click="addArtist(artist)"
              />
              <v-list-item
                v-if="!artistSearchLoading && artistResults.length === 0"
                :title="artistSearch ? $t('no_content') : $t('search_artists_hint')"
                disabled
              />
            </v-list>
          </v-window-item>

          <!-- Track seeds -->
          <v-window-item value="tracks">
            <v-text-field
              v-model="trackSearch"
              :placeholder="$t('search_tracks')"
              density="compact"
              variant="outlined"
              hide-details
              clearable
              class="mb-2"
              @update:model-value="onTrackSearchInput"
            />
            <v-list
              density="compact"
              max-height="140"
              style="overflow-y: auto"
              class="border rounded"
            >
              <v-list-item v-if="trackSearchLoading" :title="$t('loading')" disabled />
              <v-list-item
                v-for="track in trackResults"
                :key="track.uri"
                :title="track.name"
                :subtitle="track.artistName"
                :disabled="!canAddSeed(track.uri, 'track')"
                @click="addTrack(track)"
              />
              <v-list-item
                v-if="!trackSearchLoading && trackResults.length === 0"
                :title="trackSearch ? $t('no_content') : $t('search_tracks_hint')"
                disabled
              />
            </v-list>
          </v-window-item>
        </v-window>

        <!-- ── Combined seeds chips ── -->
        <div v-if="allSeeds.length" class="d-flex flex-wrap ga-1 mt-3">
          <v-chip
            v-for="seed in allSeeds"
            :key="seed.uri"
            size="small"
            closable
            @click:close="removeSeed(seed)"
          >
            <span class="opacity-50 mr-1">{{ seed.typeLabel }}</span>{{ seed.name }}
          </v-chip>
        </div>

        <!-- ── Audio feature sliders ── -->
        <div class="mt-4">
          <div class="text-subtitle-2 font-weight-medium mb-1">
            {{ $t("spotify_recommendations_features_hint") }}
          </div>
          <div class="text-caption text-medium-emphasis mb-3">
            {{ $t("spotify_recommendations_features_desc") }}
          </div>

          <div
            v-for="feature in AUDIO_FEATURES"
            :key="feature.key"
            class="d-flex align-center ga-2 mb-1"
          >
            <v-checkbox
              v-model="featureEnabled[feature.key]"
              density="compact"
              hide-details
              class="flex-shrink-0"
            />
            <span class="text-caption flex-shrink-0" style="width: 100px">
              {{ $t(feature.labelKey) }}
            </span>
            <span
              class="text-caption text-medium-emphasis text-right flex-shrink-0"
              style="width: 56px"
            >
              {{ feature.minLabel }}
            </span>
            <v-slider
              v-model="featureValue[feature.key]"
              :min="feature.min"
              :max="feature.max"
              :step="feature.step"
              :disabled="!featureEnabled[feature.key]"
              hide-details
              density="compact"
              class="flex-1"
            />
            <span class="text-caption text-medium-emphasis flex-shrink-0" style="width: 56px">
              {{ feature.maxLabel }}
            </span>
            <span
              class="text-caption text-medium-emphasis text-right flex-shrink-0"
              style="width: 56px; font-family: monospace"
            >
              {{ featureEnabled[feature.key] ? formatFeatureValue(feature, featureValue[feature.key]) : "—" }}
            </span>
          </div>
        </div>

        <!-- ── Track count ── -->
        <div class="d-flex align-center ga-3 mt-4">
          <span class="text-body-2 flex-shrink-0">{{ $t("track_count") }}</span>
          <v-select
            v-model="trackCount"
            :items="['10', '25', '50', '100']"
            density="compact"
            variant="outlined"
            hide-details
            style="max-width: 100px"
          />
        </div>

        <!-- ── Playlist name (revealed when saving) ── -->
        <v-text-field
          v-if="showSaveField"
          v-model="playlistName"
          :label="$t('new_playlist_name')"
          density="compact"
          variant="outlined"
          hide-details
          autofocus
          class="mt-4"
          @keyup.enter="doSaveAsPlaylist"
        />
      </v-card-text>

      <v-card-actions>
        <v-btn variant="text" @click="showDialog = false">{{ $t("close") }}</v-btn>
        <v-spacer />
        <v-btn
          variant="outlined"
          :disabled="totalSeeds === 0 || loading"
          @click="toggleSaveField"
        >
          <v-icon start>mdi-content-save</v-icon>
          {{ $t("settings.save") }}
        </v-btn>
        <v-btn
          v-if="showSaveField"
          color="primary"
          :disabled="!playlistName || totalSeeds === 0"
          :loading="loading"
          @click="doSaveAsPlaylist"
        >
          {{ $t("settings.save") }} →
        </v-btn>
        <v-btn
          color="primary"
          :disabled="totalSeeds === 0"
          :loading="loading"
          @click="doPlayNow"
        >
          <v-icon start>mdi-play</v-icon>
          {{ $t("play_now") }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue";
import { toast } from "vue-sonner";

import api from "@/plugins/api";
import { MediaType, QueueOption } from "@/plugins/api/interfaces";
import { type BuildSpotifyPlaylistEvent, eventbus } from "@/plugins/eventbus";
import { $t } from "@/plugins/i18n";
import router from "@/plugins/router";
import { store } from "@/plugins/store";

// -- types --

interface SeedItem {
  uri: string;
  name: string;
  artistName?: string;
}

interface AudioFeatureConfig {
  key: string;
  labelKey: string;
  min: number;
  max: number;
  step: number;
  minLabel: string;
  maxLabel: string;
  unit?: string;
  isInt?: boolean;
}

// -- static data --

const STATIC_GENRES = [
  "acoustic", "ambient", "blues", "bossanova", "brazilian", "british",
  "classical", "club", "country", "dance", "dancehall", "death-metal",
  "disco", "drum-and-bass", "dub", "dubstep", "edm", "electro", "electronic",
  "emo", "folk", "french", "funk", "gospel", "goth", "grunge", "guitar",
  "happy", "hard-rock", "heavy-metal", "hip-hop", "house",
  "indie", "indie-pop", "jazz", "k-pop", "latin", "metal", "new-age",
  "opera", "piano", "pop", "progressive-house", "punk", "punk-rock", "r-n-b",
  "reggae", "reggaeton", "rock", "rock-n-roll", "romance", "sad", "salsa",
  "samba", "singer-songwriter", "ska", "sleep", "soul",
  "study", "summer", "synth-pop", "techno", "trance", "trip-hop",
  "world-music",
];

const AUDIO_FEATURES: AudioFeatureConfig[] = [
  { key: "acousticness",     labelKey: "acousticness",     min: 0,   max: 1,   step: 0.05, minLabel: "Electronic",  maxLabel: "Acoustic" },
  { key: "danceability",     labelKey: "danceability",     min: 0,   max: 1,   step: 0.05, minLabel: "Low",         maxLabel: "High" },
  { key: "energy",           labelKey: "energy",           min: 0,   max: 1,   step: 0.05, minLabel: "Calm",        maxLabel: "Energetic" },
  { key: "instrumentalness", labelKey: "instrumentalness", min: 0,   max: 1,   step: 0.05, minLabel: "Vocal",       maxLabel: "Instrumental" },
  { key: "liveness",         labelKey: "liveness",         min: 0,   max: 1,   step: 0.05, minLabel: "Studio",      maxLabel: "Live" },
  { key: "loudness",         labelKey: "loudness",         min: -60, max: 0,   step: 1,    minLabel: "Quiet",       maxLabel: "Loud",        unit: " dB" },
  { key: "popularity",       labelKey: "popularity",       min: 0,   max: 100, step: 1,    minLabel: "Niche",       maxLabel: "Popular",     isInt: true },
  { key: "speechiness",      labelKey: "speechiness",      min: 0,   max: 1,   step: 0.05, minLabel: "Music",       maxLabel: "Spoken" },
  { key: "tempo",            labelKey: "tempo",            min: 40,  max: 220, step: 1,    minLabel: "Slow",        maxLabel: "Fast",        unit: " BPM", isInt: true },
  { key: "valence",          labelKey: "valence",          min: 0,   max: 1,   step: 0.05, minLabel: "Dark/Sad",    maxLabel: "Happy" },
];

const DEFAULT_VALUES: Record<string, number> = {
  acousticness: 0.5, danceability: 0.5, energy: 0.5,
  instrumentalness: 0.5, liveness: 0.5, loudness: -30,
  popularity: 50, speechiness: 0.5, tempo: 120, valence: 0.5,
};

// -- state --

const showDialog = ref(false);
const loading = ref(false);
const activeTab = ref("genres");

// Seeds
const selectedGenres = ref<string[]>([]);
const selectedArtists = ref<SeedItem[]>([]);
const selectedTracks = ref<SeedItem[]>([]);
const totalSeeds = computed(
  () => selectedGenres.value.length + selectedArtists.value.length + selectedTracks.value.length,
);

// Genre filter
const genreSearch = ref("");
const availableGenres = ref<string[]>(STATIC_GENRES);
const filteredGenres = computed(() => {
  const q = genreSearch.value.toLowerCase();
  return availableGenres.value.filter(
    (g) => g.includes(q) && !selectedGenres.value.includes(g),
  );
});

// Artist search
const artistSearch = ref("");
const artistResults = ref<SeedItem[]>([]);
const artistSearchLoading = ref(false);
let artistTimer: ReturnType<typeof setTimeout> | undefined;

// Track search
const trackSearch = ref("");
const trackResults = ref<SeedItem[]>([]);
const trackSearchLoading = ref(false);
let trackTimer: ReturnType<typeof setTimeout> | undefined;

// Audio features
const featureEnabled = reactive<Record<string, boolean>>(
  Object.fromEntries(AUDIO_FEATURES.map((f) => [f.key, false])),
);
const featureValue = reactive<Record<string, number>>(
  Object.fromEntries(AUDIO_FEATURES.map((f) => [f.key, DEFAULT_VALUES[f.key]])),
);

// Track count & save
const trackCount = ref("25");
const showSaveField = ref(false);
const playlistName = ref("");

// -- lifecycle --

watch(showDialog, (open) => { store.dialogActive = open; });

onMounted(async () => {
  eventbus.on("buildSpotifyPlaylist", (_evt: BuildSpotifyPlaylistEvent) => {
    selectedGenres.value = [];
    selectedArtists.value = [];
    selectedTracks.value = [];
    genreSearch.value = "";
    artistSearch.value = "";
    artistResults.value = [];
    trackSearch.value = "";
    trackResults.value = [];
    showSaveField.value = false;
    playlistName.value = "";
    loading.value = false;
    activeTab.value = "genres";
    showDialog.value = true;
  });

  // Fetch full genre list in background; static list is already shown
  try {
    const genres = await api.getSpotifyRecommendationGenres();
    if (genres.length) availableGenres.value = genres;
  } catch { /* keep static list */ }
});

onBeforeUnmount(() => {
  eventbus.off("buildSpotifyPlaylist");
  clearTimeout(artistTimer);
  clearTimeout(trackTimer);
});

// -- combined seeds view --

interface DisplaySeed {
  uri: string;
  name: string;
  typeLabel: string;
  type: "genre" | "artist" | "track";
}

const allSeeds = computed<DisplaySeed[]>(() => [
  ...selectedGenres.value.map((g) => ({ uri: g, name: g, typeLabel: $t("genre"), type: "genre" as const })),
  ...selectedArtists.value.map((a) => ({ ...a, typeLabel: $t("artist"), type: "artist" as const })),
  ...selectedTracks.value.map((t) => ({ ...t, typeLabel: $t("track"), type: "track" as const })),
]);

const canAddSeed = (uri: string, type: "artist" | "track"): boolean => {
  if (totalSeeds.value >= 5) return false;
  if (type === "artist") return !selectedArtists.value.some((a) => a.uri === uri);
  return !selectedTracks.value.some((t) => t.uri === uri);
};

const removeSeed = (seed: DisplaySeed) => {
  if (seed.type === "genre") removeGenre(seed.uri);
  else if (seed.type === "artist") removeArtist(seed.uri);
  else removeTrack(seed.uri);
};

// -- genre --

const addGenre = (genre: string) => {
  if (totalSeeds.value < 5 && !selectedGenres.value.includes(genre)) {
    selectedGenres.value.push(genre);
  }
};

const removeGenre = (genre: string) => {
  selectedGenres.value = selectedGenres.value.filter((g) => g !== genre);
};

// -- artist search --

const onArtistSearchInput = () => {
  clearTimeout(artistTimer);
  if (!artistSearch.value.trim()) { artistResults.value = []; return; }
  artistTimer = setTimeout(async () => {
    artistSearchLoading.value = true;
    try {
      const r = await api.search(artistSearch.value, [MediaType.ARTIST], 8);
      artistResults.value = r.artists.map((a) => ({ uri: a.uri, name: a.name }));
    } catch { artistResults.value = []; }
    finally { artistSearchLoading.value = false; }
  }, 350);
};

const addArtist = (artist: SeedItem) => {
  if (totalSeeds.value < 5 && !selectedArtists.value.some((a) => a.uri === artist.uri)) {
    selectedArtists.value.push(artist);
  }
};

const removeArtist = (uri: string) => {
  selectedArtists.value = selectedArtists.value.filter((a) => a.uri !== uri);
};

// -- track search --

const onTrackSearchInput = () => {
  clearTimeout(trackTimer);
  if (!trackSearch.value.trim()) { trackResults.value = []; return; }
  trackTimer = setTimeout(async () => {
    trackSearchLoading.value = true;
    try {
      const r = await api.search(trackSearch.value, [MediaType.TRACK], 8);
      trackResults.value = r.tracks.map((t) => {
        const artist = Array.isArray(t.artists) && t.artists.length ? t.artists[0] : undefined;
        return {
          uri: t.uri,
          name: t.name,
          artistName: artist && "name" in artist ? (artist as { name: string }).name : undefined,
        };
      });
    } catch { trackResults.value = []; }
    finally { trackSearchLoading.value = false; }
  }, 350);
};

const addTrack = (track: SeedItem) => {
  if (totalSeeds.value < 5 && !selectedTracks.value.some((t) => t.uri === track.uri)) {
    selectedTracks.value.push(track);
  }
};

const removeTrack = (uri: string) => {
  selectedTracks.value = selectedTracks.value.filter((t) => t.uri !== uri);
};

// -- audio features --

const formatFeatureValue = (feature: AudioFeatureConfig, value: number): string => {
  if (feature.isInt) return `${Math.round(value)}${feature.unit ?? ""}`;
  return `${value.toFixed(2)}${feature.unit ?? ""}`;
};

const buildAudioFeatures = (): Record<string, number> | undefined => {
  const result: Record<string, number> = {};
  let hasAny = false;
  for (const f of AUDIO_FEATURES) {
    if (featureEnabled[f.key]) {
      result[`target_${f.key}`] = f.isInt ? Math.round(featureValue[f.key]) : featureValue[f.key];
      hasAny = true;
    }
  }
  return hasAny ? result : undefined;
};

// -- save / play --

const toggleSaveField = () => {
  showSaveField.value = !showSaveField.value;
};

const doPlayNow = async () => {
  if (totalSeeds.value === 0) return;
  const queueId =
    store.activePlayer?.active_source && store.activePlayer.active_source in api.queues
      ? store.activePlayer.active_source
      : store.activePlayer?.player_id;
  if (!queueId) { toast.error($t("no_player")); return; }
  loading.value = true;
  try {
    await api.getSpotifyRecommendations(
      selectedTracks.value.map((t) => t.uri),
      selectedArtists.value.map((a) => a.uri),
      selectedGenres.value,
      Number(trackCount.value),
      buildAudioFeatures(),
      queueId,
      QueueOption.REPLACE,
    );
    showDialog.value = false;
  } catch { toast.error($t("spotify_recommendations_error")); }
  finally { loading.value = false; }
};

const doSaveAsPlaylist = async () => {
  if (!playlistName.value || totalSeeds.value === 0) return;
  loading.value = true;
  try {
    const tracks = await api.getSpotifyRecommendations(
      selectedTracks.value.map((t) => t.uri),
      selectedArtists.value.map((a) => a.uri),
      selectedGenres.value,
      Number(trackCount.value),
      buildAudioFeatures(),
    );
    // Use the builtin provider so tracks are stored by URI in MA's DB and appear immediately.
    const playlist = await api.createPlaylist(playlistName.value);
    await api.addPlaylistTracks(playlist.item_id, tracks.map((t) => t.uri));
    showDialog.value = false;
    toast.success($t("playlist_created"), {
      action: {
        label: $t("open_playlist"),
        onClick: () => {
          store.showFullscreenPlayer = false;
          router.push({ name: "playlist", params: { itemId: playlist.item_id, provider: playlist.provider } });
        },
      },
    });
  } catch { toast.error($t("spotify_recommendations_error")); }
  finally { loading.value = false; }
};
</script>
