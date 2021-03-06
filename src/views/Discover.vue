<template>
  <div class="discover">
    <div class="container">
      <form @submit.prevent="searchRequest">
        <div class="discover__form">
          <div class="discover__form-field-container">
            <div>{{ "discover-mediatype" | localize }}</div>
            <select class="discover__form-field" v-model="media_type">
              <option
                v-for="mediatype in mediatypeOptions"
                :value="mediatype.value"
                :key="mediatype.value"
                >{{ mediatype.title }}</option
              >
            </select>
          </div>
          <div class="discover__form-field-container">
            <div>{{ "discover-sort" | localize }}</div>
            <select class="discover__form-field" v-model="sortType">
              <option
                v-for="sort in sortOptions"
                :value="sort.value"
                :key="sort.value"
                >{{ sort.title }}</option
              >
            </select>
          </div>
          <div class="discover__form-field-container">
            <div>{{ "discover-min-average-vote" | localize }}</div>
            <input
              class="discover__form-field"
              type="number"
              v-model="vote"
              :placeholder="minAverageVotePlaceholder"
            />
          </div>
          <div
            v-if="media_type === 'movie'"
            class="discover__form-field-container"
          >
            <div>{{ "discover-involved-actor" | localize }}</div>
            <autocomplete
              v-if="!actor.name"
              :search="search"
              :get-result-value="getResultValue"
              @submit="onSubmit"
              auto-select
            />
            <input
              v-else-if="actor.name"
              class="discover__form-field"
              :value="actor.name"
              @focus="actor.name = null"
            />
          </div>
          <div class="discover__form-field-container">
            <div>{{ "discover-genre" | localize }}</div>
            <select class="discover__form-field" v-model="genre">
              <option value=""></option>
              <option v-for="genre in genres" :key="genre.id">{{
                genre.name
              }}</option>
            </select>
          </div>
          <div class="discover__form-field-container">
            <div>
              {{
                media_type === "movie"
                  ? "discover-year"
                  : "discover-first-airing-date" | localize
              }}
            </div>
            <input
              class="discover__form-field"
              type="number"
              v-model="year"
              :placeholder="yearPlaceholder"
            />
          </div>
        </div>
        <div class="discover__form-button-container">
          <button class="discover__form-button" type="submit">
            {{ "discover-submit" | localize }}
          </button>
        </div>
      </form>
      <div class="discover__results" v-if="searchResultPage.total_results">
        <div
          class="discover__result"
          v-for="item in searchResultPage.results"
          :key="item.id"
        >
          <CoverTemplate1 :item="item" />
        </div>
      </div>
      <div class="discover__results-title" v-else>
        {{ "discover-nothing-found" | localize }}
      </div>
      <div v-if="searchResultPage.page" class="discover__page-buttons">
        <button
          class="discover__page-button discover__page-button--previous"
          v-if="searchResultPage.page > 1"
          @click="getPreviousPageSearchResults()"
        >
          {{ "discover-previous" | localize }}
        </button>
        <button
          class="discover__page-button discover__page-button--next"
          v-if="searchResultPage.page < searchResultPage.total_pages"
          @click="getNextPageSearchResults()"
        >
          {{ "discover-next" | localize }}
        </button>
      </div>
    </div>
  </div>
</template>
<script>
import axios from "axios";
import CoverTemplate1 from "../components/CoverTemplate1.vue";
import Autocomplete from "@trevoreyre/autocomplete-vue";
import localize from "@/filters/localize";

export default {
  data() {
    return {
      movieSortOptions: [],
      tvSortOptions: [],
      mediatypeOptions: [],
      minAverageVotePlaceholder: "",
      yearPlaceholder: "",
      year: "",
      genre: "",
      people: "",
      vote: "",
      sortType: "",
      media_type: "",
      searchResultPage: {},
      movieGenres: null,
      tvshowGenres: null,
      searchQuery: "",
      pageNumber: "",
      fullPath: "",
      selectedActorIDFromList: "",
      selectedGenre: "",
      selectedGenreID: "",
      selectedActor: "",
      selectedVote: "",
      routeMediatype: "",
      routeSortBy: "",
      routeVote: "",
      routeActorID: "",
      routeGenreID: "",
      routeYear: "",
      routePage: "",
      key: process.env.VUE_APP_MOVIEDB,
      actor: {},
      preventOnCreatedUpdate: true,
      locale: "",
      arr2: [],
      selectedActorID: '',
      selectedActorName: ''
    };
  },
  components: {
    CoverTemplate1,
    Autocomplete,
  },
  computed: {
    genres() {
      return this.media_type === "movie" ? this.movieGenres : this.tvshowGenres;
    },
    sortOptions() {
      return this.media_type === "movie"
        ? this.movieSortOptions
        : this.tvSortOptions;
    },
  },
  watch: {
    media_type() {
      if (this.preventOnCreatedUpdate) {
        this.preventOnCreatedUpdate = false;
        return;
      }
      this.genre = this.routeGenreID = this.selectedGenreID = "";
      this.selectedActorIDFromList = this.routeActorID = "";
      this.actor = {};
      this.routeSortBy = this.sortType = "popularity.desc";
    },
    $route() {
      this.getRoutePaths();
      if(this.routeMediatype) {
        this.media_type = "";
        this.media_type = this.routeMediatype;
      }
      if(this.routeVote) {
        this.vote = "";
        this.vote = this.routeVote;
      } else if(!this.routeVote) {
        this.vote = "";
      }
      if(this.routeYear) {
        this.year = "";
        this.year = this.routeYear;
      } else if(!this.routeYear) {
        this.year = "";
      }
      if(this.routeSortBy) {
        this.sortType = "";
        this.sortType = this.routeSortBy;
      }
      if (this.routeGenreID) {
        this.genre = ''
        this.selectedGenre = this.genres.find(
          (name) => name.id === +this.routeGenreID
        );
        this.genre = this.selectedGenre.name;
      } else if (!this.routeGenreID) {
        this.genre = ''
      }
      if (this.routeActorID) {
        this.actor = {}
        axios
          .get(
            `${this.globalAPIMovieDBAddress}/3/person/${this.routeActorID}?api_key=${this.key}&&language=${this.locale}`
          )
          .then((response) => {
            this.actor = response.data;
          });
      } else if (!this.routeActorID) {
        this.actor = {}
      }
      // this.routeGenreID = "";
      // this.routeActorID = "";
      if (this.isSearchQueryCorrect()) {
        this.getLocalizedSelectsValues()
        
        this.getPageSearchResults(
          this.routeMediatype,
          this.routeSortBy,
          this.routeVote,
          this.routeActorID,
          this.routeGenreID,
          this.routeYear,
          this.routePage
        );
      } else {
        this.searchResultPage = {};
        this.$root.loading = false;
      }
    },
    "$store.getters.locale"() {
      this.changeLocale();
    },
  },
  async created() {
    this.locale = this.$store.getters.locale;
    this.getRoutePaths();
    if(this.routeMediatype) {
      this.media_type = "";
      this.media_type = this.routeMediatype;
    }
    if(this.routeVote) {
      this.vote = "";
      this.vote = this.routeVote;
    }
    if(this.routeYear) {
      this.year = "";
      this.year = this.routeYear;
    }
    if(this.routeSortBy) {
      this.sortType = "";
      this.sortType = this.routeSortBy;
    }
    this.movieGenres = this.$store.getters.MovieGenres;
    this.tvshowGenres = this.$store.getters.TVShowGenres;
    if (this.routeGenreID) {
      this.genre = ''
      this.selectedGenre = this.genres.find(
        (name) => name.id === +this.routeGenreID
      );
      this.genre = this.selectedGenre.name;
    }
    if (this.routeActorID) {
      this.actor = {}
      await axios
        .get(
          `${this.globalAPIMovieDBAddress}/3/person/${this.routeActorID}?api_key=${this.key}&&language=${this.locale}`
        )
        .then((response) => {
          this.actor = response.data;
        });
    }
    this.getLocalizedSelectsValues()
    if (this.isSearchQueryCorrect()) {
      // this.media_type = this.routeMediatype;
      // this.vote = this.routeVote;
      // this.year = this.routeYear;
      // this.sortType = this.routeSortBy;
      this.getPageSearchResults(
        this.routeMediatype,
        this.routeSortBy,
        this.routeVote,
        this.routeActorID,
        this.routeGenreID,
        this.routeYear,
        this.routePage
      );

      
    } else {
      this.searchResultPage = {};
      this.$root.loading = false;
    }
  },
  methods: {
    isSearchQueryCorrect() {
      let arr = Object.keys(this.$route.query);
      if (
        this.$route.query.mediatype &&
        this.$route.query.mediatype === "movie"
      ) {
        this.arr2 = [
          "mediatype",
          "page",
          "sort_by",
          "vote_average",
          "with_genres",
          "with_people",
          "year",
        ];
        return (
          this.arr2.every((i) => arr.includes(i)) &&
          this.movieSortOptions.some(
            (i) => i.value === this.$route.query.sort_by
          ) &&
          this.$route.query.page && +this.$route.query.page !== 0
        );
      } else if (
        this.$route.query.mediatype &&
        this.$route.query.mediatype === "tv"
      ) {
        this.arr2 = [
          "mediatype",
          "page",
          "sort_by",
          "vote_average",
          "with_genres",
          "first_air_date_year",
        ];
        return (
          this.arr2.every((i) => arr.includes(i)) &&
          this.tvSortOptions.some(
            (i) => i.value === this.$route.query.sort_by
          ) &&
          this.$route.query.page && +this.$route.query.page !== 0
        );
      }
    },
    getLocalizedSelectsValues() {
      this.yearPlaceholder = localize("discover-year");
      this.minAverageVotePlaceholder = localize("discover-min-average-vote");
      this.mediatypeOptions = [
        { title: localize("discover-mediatype-movie"), value: "movie" },
        { title: localize("discover-mediatype-tvshow"), value: "tv" },
      ];
      this.movieSortOptions = [
        {
          title: localize("discover-popularity-descending"),
          value: "popularity.desc",
        },
        {
          title: localize("discover-popularity-ascending"),
          value: "popularity.asc",
        },
        {
          title: localize("discover-release-date-descending"),
          value: "release_date.desc",
        },
        {
          title: localize("discover-release-date-ascending"),
          value: "release_date.asc",
        },
        {
          title: localize("discover-revenue-descending"),
          value: "revenue.desc",
        },
        {
          title: localize("discover-revenue-ascending"),
          value: "revenue.asc",
        },
        {
          title: localize("discover-vote-average-descending"),
          value: "vote_average.desc",
        },
        {
          title: localize("discover-vote-average-ascending"),
          value: "vote_average.asc",
        },
      ];
      this.tvSortOptions = [
        {
          title: localize("discover-popularity-descending"),
          value: "popularity.desc",
        },
        {
          title: localize("discover-popularity-ascending"),
          value: "popularity.asc",
        },
        {
          title: localize("discover-first-air-date-descending"),
          value: "first_air_date.desc",
        },
        {
          title: localize("discover-first-air-date-ascending"),
          value: "first_air_date.asc",
        },
        {
          title: localize("discover-vote-average-descending"),
          value: "vote_average.desc",
        },
        {
          title: localize("discover-vote-average-ascending"),
          value: "vote_average.asc",
        },
      ];
    },
    async changeLocale() {
      this.locale = this.$store.getters.locale;
      this.getLocalizedSelectsValues()
      this.getRoutePaths();
      // this.media_type = "";
      // this.media_type = this.routeMediatype;
      // this.vote = "";
      // this.vote = this.routeVote;
      // this.year = "";
      // this.year = this.routeYear;
      // this.sortType = "";
      // this.sortType = this.routeSortBy;
      // this.routeGenreID = "";
      // this.routeActorID = "";
      if (this.isSearchQueryCorrect()) {
        this.getLocalizedSelectsValues()
        
        this.getPageSearchResults(
          this.routeMediatype,
          this.routeSortBy,
          this.routeVote,
          this.routeActorID,
          this.routeGenreID,
          this.routeYear,
          this.routePage
        );
      } else {
        this.searchResultPage = {};
        this.$root.loading = false;
      }
      await this.$store.dispatch("loadMovieGenres");
      this.movieGenres = this.$store.getters.MovieGenres;
      await this.$store.dispatch("loadTVShowsGenres");
      this.tvshowGenres = this.$store.getters.TVShowGenres;
      if (this.routeGenreID) {
        this.selectedGenre = this.genres.find(
          (name) => name.id === +this.routeGenreID
        );
        this.genre = this.selectedGenre.name;
      }
      if (this.routeActorID) {
        await axios
          .get(
            `${this.globalAPIMovieDBAddress}/3/person/${this.routeActorID}?api_key=${this.key}&&language=${this.locale}`
          )
          .then((response) => {
            this.actor = response.data;
          });
      }
    },
    getFullPath() {
      this.fullPath = this.$route.fullPath.split("page=")[0];
    },
    increasePageNumber() {
      this.pageNumber = this.$route.query.page;
      this.pageNumber++;
    },
    decreasePageNumber() {
      this.pageNumber = this.$route.query.page;
      this.pageNumber--;
    },
    getRoutePaths() {
      this.routeMediatype = this.$route.query.mediatype;
      this.routeSortBy = this.$route.query.sort_by;
      this.routeVote = this.$route.query.vote_average;
      this.routeActorID = this.$route.query.with_people;
      this.routeGenreID = this.$route.query.with_genres;
      this.routeYear =
        this.routeMediatype === "movie"
          ? this.$route.query.year
          : this.$route.query.first_air_date_year;
      this.routePage = this.$route.query.page;
    },
    async search(input) {
      if (input.length < 1) {
        this.selectedActorIDFromList = "";
        return [];
      }
      let a = [];
      await axios
        .get(
          `${this.globalAPIMovieDBAddress}/3/search/person?api_key=${this.key}&language=${this.locale}&query=${input}&include_adult=false&page=1`
        )
        .then((response) => {
          a = response.data.results;
          a.filter((actor) => {
            return actor.name.toLowerCase().includes(input.toLowerCase());
          });
        });
      this.$root.loading = false;
      return a;
    },
    getResultValue(result) {
      return result.name;
    },
    onSubmit(result) {
      if (result) {
        this.selectedActorIDFromList = result.id;
      }
    },
    searchRequest() {
      if (this.genre) {
        this.selectedGenre = this.genres.find(
          (name) => name.name === this.genre
        );
        this.selectedGenreID = this.selectedGenre.id;
      } else if (!this.genre) {
        this.selectedGenreID = "";
      }
      if (this.media_type === "tv") {
        this.selectedActor = "";
      }
      if (this.selectedActorIDFromList || this.actor.id) {
        this.selectedActor = this.selectedActorIDFromList || this.actor.id;
      }
      if (this.media_type === "movie") {
        this.$router.push(
          `${this.$route.path}?mediatype=${this.media_type}&sort_by=${this.sortType}&vote_average=${this.vote}&with_people=${this.selectedActor}&with_genres=${this.selectedGenreID}&year=${this.year}&page=1`
        );
      } else if (this.media_type === "tv") {
        this.$router.push(
          `${this.$route.path}?mediatype=${this.media_type}&sort_by=${this.sortType}&vote_average=${this.vote}&with_genres=${this.selectedGenreID}&first_air_date_year=${this.year}&page=1`
        );
      }
    },
    async getPageSearchResults(
      routeMediatype,
      routeSortBy,
      routeVote,
      routeActorID,
      routeGenreID,
      routeYear,
      routePage
    ) {
      if (routeMediatype === "movie") {
        await axios
          .get(
            `${this.globalAPIMovieDBAddress}/3/discover/${routeMediatype}?api_key=${this.key}&language=${this.locale}&sort_by=${routeSortBy}&include_adult=false&include_video=false&vote_average.gte=${routeVote}&with_people=${routeActorID}&with_genres=${routeGenreID}&year=${routeYear}&page=${routePage}`
          )
          .then((response) => {
            this.searchResultPage = response.data;
            if (this.searchResultPage.results.length) {
              for (let i = 0; i < this.searchResultPage.results.length; i++) {
                this.$set(
                  this.searchResultPage.results[i],
                  "media_type",
                  routeMediatype
                );
              }
            }
          });
      } else if (routeMediatype === "tv") {
        await axios
          .get(
            `${this.globalAPIMovieDBAddress}/3/discover/${routeMediatype}?api_key=${this.key}&language=${this.locale}&sort_by=${routeSortBy}&include_adult=false&include_video=false&vote_average.gte=${routeVote}&with_genres=${routeGenreID}&first_air_date_year=${routeYear}&page=${routePage}`
          )
          .then((response) => {
            this.searchResultPage = response.data;
            if (this.searchResultPage.results.length) {
              for (let i = 0; i < this.searchResultPage.results.length; i++) {
                this.$set(
                  this.searchResultPage.results[i],
                  "media_type",
                  routeMediatype
                );
              }
            }
          });
      }
      this.$root.loading = false;
    },
    getNextPageSearchResults() {
      this.getFullPath();
      this.increasePageNumber();
      this.$router.push(`${this.fullPath}&page=${this.pageNumber}`);
    },
    getPreviousPageSearchResults() {
      this.getFullPath();
      this.decreasePageNumber();
      this.$router.push(`${this.fullPath}&page=${this.pageNumber}`);
    },
  },
};
</script>
<style scoped>
.discover {
  background: var(--main-bg);
}
.discover__form {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
}
.discover__form-field-container {
  display: flex;
  flex-direction: column;
  margin: 10px;
}
@media (min-width: 640px) {
  .discover__form-field-container {
    width: 150px;
  }
}
@media (min-width: 480px) and (max-width: 639px) {
  .discover__form-field-container {
    width: 170px;
  }
}
@media (max-width: 479px) {
  .discover__form-field-container {
    width: 280px;
  }
}
.discover__form-field {
  outline: none;
  border: none;
  margin-top: 1px;
}
.discover__form-button-container {
  display: flex;
  justify-content: center;
}
.discover__form-button,
.discover__page-button {
  background: none;
  border: 2px solid var(--main-text-color);
  border-radius: 20px;
  cursor: pointer;
  font-family: "Alegreya Sans", sans-serif;
  font-size: 1rem;
  letter-spacing: 0.1rem;
  margin: 2rem auto;
  outline: none;
  padding: 0.7rem;
  width: 180px;
}
.discover__results {
  display: flex;
  flex-wrap: wrap;
}
.discover__result {
  display: flex;
  flex-direction: column;
  height: auto;
}
@media (min-width: 1280px) {
  .discover__result {
    width: 20%;
  }
}
@media (min-width: 1024px) and (max-width: 1279px) {
  .discover__result {
    width: 25%;
  }
}
@media (min-width: 768px) and (max-width: 1023px) {
  .discover__result {
    width: 33%;
  }
}
@media (min-width: 480px) and (max-width: 767px) {
  .discover__result {
    width: 50%;
  }
}
@media (max-width: 479px) {
  .discover__result {
    width: 100%;
  }
}
.discover__results-title {
  display: flex;
  justify-content: center;
}
.discover__page-buttons {
  display: flex;
  justify-content: center;
  margin-top: 36px;
}
.discover__page-button--previous::before {
  content: "<<";
}
.discover__page-button--next::after {
  content: ">>";
}
</style>
