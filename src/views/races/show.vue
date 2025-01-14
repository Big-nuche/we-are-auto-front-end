<template>
  <div class="p-4 relative" v-if="race.length && races.length">
    <div class="my-2 ml-2 mr-20">
      <v-card
        :heading="race.name"
        :subheading="race.track.name"
        class="row-span-2 col-span-3"
      >
        <div
          v-show="raceState(race) === 'video'"
          id="wrapper"
          class="videoWrapper"
        >
          <div id="player"></div>
        </div>
        <div v-show="raceState(race) === 'add'">
          <!-- Check if authenticated here -->
          <suggestion :race="race" />
        </div>

        <div v-show="raceState(race) === 'locked'">
          <div>
            The {{ race.name }} is scheduled to start on
            {{ humanReadableDate(race) }}
          </div>
        </div>

        <nav aria-label="Progress">
          <ol
            role="list"
            class="divide-y divide-gray-300 md:flex md:divide-y-0"
          >
            <li
              v-for="(video, index) in race.videos"
              :key="video.id"
              class="relative md:flex-1 md:flex"
              @click="selectVideoPart(index)"
            >
              <div class="group flex items-center w-full cursor-pointer">
                <span
                  class="px-6 py-4 flex items-center text-sm font-medium group"
                >
                  <span
                    class="
                      flex-shrink-0
                      w-10
                      h-10
                      flex
                      items-center
                      justify-center
                      bg-white
                      shadow-md
                      border
                      rounded-full
                      group-hover:bg-blue-500
                    "
                  >
                    <video-progress :video="video" />
                  </span>
                  <span
                    class="
                      ml-4
                      text-sm
                      font-medium
                      text-gray-500
                      group-hover:text-gray-700
                    "
                    >Part {{ index + 1 }}</span
                  >
                </span>
              </div>
              <template v-if="index !== race.videos.length - 1">
                <!-- Arrow separator for lg screens and up -->
                <div
                  class="hidden md:block absolute top-0 right-0 h-full w-5"
                  aria-hidden="true"
                >
                  <svg
                    class="h-full w-full text-gray-300"
                    viewBox="0 0 22 80"
                    fill="none"
                    preserveAspectRatio="none"
                  >
                    <path
                      d="M0 -2L20 40L0 82"
                      vector-effect="non-scaling-stroke"
                      stroke="currentcolor"
                      stroke-linejoin="round"
                    />
                  </svg>
                </div>
              </template>
            </li>
          </ol>
        </nav>
      </v-card>
    </div>

    <div
      class="
        absolute
        right-0
        top-0
        mt-6
        z-10
        px-4
        transform
        translate-x-full
        mx-22
      "
    >
      <div
        @mouseover="seasonOpen = true"
        @mouseleave="seasonOpen = false"
        :class="[
          seasonOpen
            ? 'transform duration-500 -translate-x-full ease-in-out mx-12'
            : 'transform duration-500 translate-x-0 ease-in-out',
        ]"
      >
        <v-card>
          <template v-slot:header>
            <div class="flex items-center p-2">
              <div class="flex-shrink-0">
                <div class="px-2">
                  <span class="sr-only">{{ race.series.name }}</span>
                  <img
                    class="h-10 w-10 rounded-full shadow"
                    :src="race.series.logo"
                    alt=""
                  />
                </div>
              </div>
              <div class="ml-3">
                <div class="text-sm font-medium text-gray-900">
                  <div class="font-bold text-gray-800">
                    {{ race.series.name }}
                  </div>
                </div>
                <div class="flex space-x-1 text-sm text-gray-400">
                  <span>{{ race.season.name }}</span>
                </div>
              </div>
            </div>
          </template>

          <div class="flow-root">
            <div class="m-4" v-if="races.length">
              <div v-for="(race, index) in races" :key="race.id">
                <router-link
                  class="group"
                  :to="{ name: 'races.show', params: { id: race.id } }"
                >
                  <div class="relative mb-8">
                    <span
                      v-if="index !== races.length - 1"
                      class="absolute top-12 left-5 h-6 w-0.5 bg-gray-200"
                      aria-hidden="true"
                    />
                    <div class="relative flex space-x-3">
                      <div>
                        <play-progress :race="race" />
                      </div>
                      <div
                        class="
                          min-w-0
                          flex-1
                          pt-1.5
                          flex
                          justify-between
                          space-x-4
                        "
                      >
                        <div class="ml-1">
                          <div
                            class="
                              text-sm text-gray-600
                              group-hover:text-gray-900
                            "
                            :class="
                              $route.params.id == race.id ? 'font-bold' : ''
                            "
                          >
                            {{ race.name }}
                          </div>
                          <div
                            class="
                              text-sm text-gray-400
                              group-hover:text-gray-700
                            "
                            :class="
                              $route.params.id == race.id ? 'font-bold' : ''
                            "
                          >
                            {{ race.track.name }}
                          </div>
                        </div>
                      </div>
                    </div>
                  </div>
                </router-link>
              </div>
            </div>
          </div>
        </v-card>
      </div>
    </div>
  </div>
</template>

<script>
import Race from "@/api/models/races.js";
import Series from "@/api/models/series.js";
import PlayProgress from "@/components/playProgress.vue";
import VideoProgress from "@/components/videoProgress.vue";
import Suggestion from "@/components/races/suggestion.vue";
export default {
  components: {
    PlayProgress,
    VideoProgress,
    Suggestion,
  },
  data() {
    return {
      race: [],
      races: [],
      player: null,
      seasonOpen: false,
      suggestionLink: null,
      validated: false,
    };
  },
  async mounted() {
    await this.getRaceData();
    window.addEventListener("resize", this.resizePlayer);
    this.initPlayer();
  },
  async unmounted() {
    // Save the video progress on leave
    await new VideoProgress().store({
      video_id: 1,
      seconds: 100,
    });
  },
  methods: {
    initPlayer() {
      if (!this.race?.videos[0]?.video_id) {
        return;
      }

      var tag = document.createElement("script");
      tag.src = "https://www.youtube.com/iframe_api";
      var firstScriptTag = document.getElementsByTagName("script")[0];
      firstScriptTag.parentNode.insertBefore(tag, firstScriptTag);
      window.onYouTubeIframeAPIReady = () => {
        /* eslint-disable-next-line */
        this.player = new YT.Player("player", {
          height: "349",
          width: "100%",
          videoId: this.race.videos[0].video_id,
          playerVars: {
            playsinline: 0,
          },
          events: {
            onReady: this.onPlayerReady,
            //'onStateChange': this.onPlayerStateChange
          },
        });
      };
    },
    onPlayerReady() {
      this.resizePlayer();
      //this.player.playVideo();
    },
    resizePlayer() {
      const width = document.getElementById("wrapper").offsetWidth;
      const height = width / (640 / 360);
      this.player.setSize(width, height);
    },
    async getRaceData() {
      ({ data: this.race } = await new Race().show(this.$route.params.id));
      ({ data: this.races } = await new Series().series_season(
        this.race?.series?.id,
        this.race?.season?.id
      ));
    },
    selectVideoPart(index) {
      // Check if you are already watching that part
      if (
        this.race.videos[index].video_id ===
        this.player.getVideoData().player_id
      ) {
        return;
      }
      this.player.cueVideoById(this.race.videos[index].video_id);
    },
    raceState(race) {
      if (race?.videos?.length >= 1) {
        return "video";
      }

      const today = new Date().getTime();
      const race_starts_at = isNaN(Date.parse(race?.starts_at))
        ? 0
        : Date.parse(race?.starts_at);

      if (today < race_starts_at) {
        return "locked";
      }

      if (!(race?.videos?.length >= 1)) {
        return "add";
      }

      return null;
    },
    humanReadableDate(race) {
      return isNaN(Date.parse(race?.starts_at))
        ? 0
        : new Date(Date.parse(race?.starts_at)).toLocaleDateString();
    },
  },
  watch: {
    async "$route.params.id"() {
      await this.getRaceData();
      if (!this.race?.videos[0]?.video_id) {
        return;
      }
      this.player.cueVideoById(this.race?.videos[0]?.video_id);
    },
  },
};
</script>

<style scoped>
.videoWrapper {
  position: relative;
  padding-bottom: 56.25%; /* 16:9 */
  height: 0;
}
.videoWrapper iframe {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.mx-22 {
  margin-left: 5.5rem;
  margin-right: 5.5rem;
}
</style>
