<template>
  <div id="app">
    <div
      v-if="player.playing"
      class="now-playing"
      :class="getNowPlayingClass()"
    >
      <div class="now-playing__cover">
        <img
          :src="player.trackAlbum.image"
          :alt="player.trackTitle"
          class="now-playing__image"
        />
      </div>
      <div class="now-playing__details">
        <h1 class="now-playing__track" v-text="player.trackTitle"></h1>
        <h2 class="now-playing__artists" v-text="getTrackArtists"></h2>
        <img
          v-if="player.trackAdderPic"
          :src="player.trackAdderPic"
          :alt="player.trackAdder"
          class="now-playing__profilepic"
        />
        <h2 class="now-playing__source" v-text="getTrackSource"></h2>
      </div>
    </div>
    <div v-else class="now-playing" :class="getNowPlayingClass()">
      <h1 class="now-playing__idle-heading">No music is playing 😔</h1>
    </div>
  </div>
</template>

<script>
import * as Vibrant from 'node-vibrant'

import props from '@/utils/props.js'

export default {
  name: 'NowPlaying',

  props: {
    auth: props.auth,
    endpoints: props.endpoints,
    player: props.player
  },

  data() {
    return {
      pollPlaying: '',
      responses: {},
      playerData: this.getEmptyPlayer(),
      colourPalette: '',
      swatches: []
    }
  },

  computed: {
    /**
     * Return a comma-separated list of track artists.
     * @return {String}
     */
    getTrackArtists() {
      return this.player.trackArtists.join(', ')
    },
    
    getTrackSource() {
      if (this.player.trackSource == "playlist") {
        return `Added by ${this.player.trackAdder}`
      }
      return `Playing from ${this.player.trackSource}`
    }
  },

  mounted() {
    this.setDataInterval()
  },

  beforeDestroy() {
    clearInterval(this.pollPlaying)
  },

  methods: {
    /**
     * Make the network request to Spotify to
     * get the current played track.
     */
    async getNowPlaying() {
      let data = {}
      let responses = {}

      try {
        const response = await fetch(
          `${this.endpoints.base}/${this.endpoints.nowPlaying}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!response.ok) {
          throw new Error(`An error has occured: ${response.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
         
        // TODO: integrate this into the watcher
        if (response.status === 204) {
          data = this.getEmptyPlayer()
          this.playerData = data
          responses = this.responses

          this.$nextTick(() => {
            this.$emit('spotifyTrackUpdated', data)
          })

          return
        }

        const playerData = await response.json()
        if (playerData.item?.id === this.playerData.trackId) {
          responses = this.responses
          return
        }
        
        /**
         * We've got a new track. Save the data and find its source.
         */
        responses.playerResponse = playerData
        const stateResponse = await fetch(
          `${this.endpoints.base}/${this.endpoints.playState}`,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!stateResponse.ok) {
          throw new Error(`An error has occured: ${stateResponse.status}`)
        }

        /**
         * Spotify returns a 204 when no current device session is found.
         * The connection was successful but there's no content to return.
         */
        if (stateResponse.status === 204) {
          return
        }

        const stateData = await stateResponse.json()
        responses.playerStateResponse = stateData
        if (stateData?.context?.type != "playlist" || stateData?.context?.href === undefined) {
          return
        }
        
                
        /**
         * We're listening to a playlist. Find out who added the current track.
         */
        const playlistResponse = await fetch(
          stateData.context.href,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!playlistResponse.ok) {
          throw new Error(`An error has occured: ${playlistResponse.status}`)
        }

        const playlistData = await playlistResponse.json()
        responses.playlistResponse = playlistData
        
        /**
         * Figure out the track adder.
         */
        var adderUrl = ""
        if (playlistData.tracks?.items?.forEach != undefined) {
          playlistData.tracks?.items?.forEach((item) => {
            if (item.track?.id === this.playerResponse.item?.id) {
              responses.addedBy = item.added_by?.id
              adderUrl = item.added_by?.href
            }
          })
        }
        if (!adderUrl) {
          return
        }
        
        /**
         * Next, fetch the URL of the adding user's profile picture.
         */
        const userProfileResponse = await fetch(
          adderUrl,
          {
            headers: {
              Authorization: `Bearer ${this.auth.accessToken}`
            }
          }
        )

        /**
         * Fetch error.
         */
        if (!userProfileResponse.ok) {
          throw new Error(`An error has occured: ${userProfileResponse.status}`)
        }

        const userProfileData = await userProfileResponse.json()
        responses.userProfileResponse = userProfileData
        responses.addedByImage = userProfileData?.images?.url
      } catch (error) {
        this.handleExpiredToken()

        data = this.getEmptyPlayer()
        this.playerData = data
        responses = this.responses

        this.$nextTick(() => {
          this.$emit('spotifyTrackUpdated', data)
        })
      } finally {
        this.responses = responses
      }
    },

    /**
     * Get the Now Playing element class.
     * @return {String}
     */
    getNowPlayingClass() {
      const playerClass = this.player.playing ? 'active' : 'idle'
      return `now-playing--${playerClass}`
    },

    /**
     * Get the colour palette from the album cover.
     */
    getAlbumColours() {
      /**
       * No image (rare).
       */
      if (!this.player.trackAlbum?.image) {
        return
      }

      /**
       * Run node-vibrant to get colours.
       */
      Vibrant.from(this.player.trackAlbum.image)
        .quality(1)
        .clearFilters()
        .getPalette()
        .then(palette => {
          this.handleAlbumPalette(palette)
        })
    },

    /**
     * Return a formatted empty object for an idle player.
     * @return {Object}
     */
    getEmptyPlayer() {
      return {
        playing: false,
        trackAlbum: {},
        trackArtists: [],
        trackId: '',
        trackTitle: ''
      }
    },

    /**
     * Poll Spotify for data.
     */
    setDataInterval() {
      clearInterval(this.pollPlaying)
      this.pollPlaying = setInterval(() => {
        this.getNowPlaying()
      }, 2500)
    },

    /**
     * Set the stylings of the app based on received colours.
     */
    setAppColours() {
      document.documentElement.style.setProperty(
        '--color-text-primary',
        this.colourPalette.text
      )

      document.documentElement.style.setProperty(
        '--colour-background-now-playing',
        this.colourPalette.background
      )
    },

    /**
     * Handle newly updated Spotify Tracks.
     */
    handleNowPlaying() {
      if (
        this.playerResponse?.error?.status === 401 ||
        this.playerResponse?.error?.status === 400 ||
        this.playerStateResponse?.error?.status === 401 ||
        this.playerStateResponse?.error?.status === 400 ||
        this.playlistResponse?.error?.status === 401 ||
        this.playlistResponse?.error?.status === 400 ||
        this.userProfileResponse?.error?.status === 401 ||
        this.userProfileResponse?.error?.status === 400
      ) {
        this.handleExpiredToken()

        return
      }

      /**
       * Player is active, but user has paused.
       */
      if (!this.responses.playerResponse?.is_playing) {
        this.playerData = this.getEmptyPlayer()

        return
      }

      /**
       * Store the current active track.
       */
      this.playerData = {
        playing: this.responses.playerResponse.is_playing,
        trackArtists: this.responses.playerResponse.item.artists.map(
          artist => artist.name
        ),
        trackTitle: this.responses.playerResponse.item.name,
        trackId: this.responses.playerResponse.item.id,
        trackAlbum: {
          title: this.responses.playerResponse.item.album.name,
          image: this.responses.playerResponse.item.album.images[0].url
        },
        trackSource: this.responses.playerStateResponse?.context?.type,
        trackAdder: this.responses.userProfileResponse?.display_name || this.responses.addedBy,
        trackAdderPic: (this.responses.userProfileResponse?.images ? this.responses.userProfileResponse?.images[0].url : "")
      }
    },

    /**
     * Handle newly stored colour palette:
     * - Map data to readable format
     * - Get and store random colour combination.
     */
    handleAlbumPalette(palette) {
      let albumColours = Object.keys(palette)
        .filter(item => {
          return item === null ? null : item
        })
        .map(colour => {
          return {
            text: palette[colour].getTitleTextColor(),
            background: palette[colour].getHex()
          }
        })

      this.swatches = albumColours

      this.colourPalette =
        albumColours[Math.floor(Math.random() * albumColours.length)]

      this.$nextTick(() => {
        this.setAppColours()
      })
    },

    /**
     * Handle an expired access token from Spotify.
     */
    handleExpiredToken() {
      clearInterval(this.pollPlaying)
      this.$emit('requestRefreshToken')
    }
  },
  watch: {
    /**
     * Watch the auth object returned from Spotify.
     */
    auth: function(oldVal, newVal) {
      if (newVal.status === false) {
        clearInterval(this.pollPlaying)
      }
    },

    /**
     * Watch the returned track object.
     */
    playerResponse: function() {
      this.handleNowPlaying()
    },
    
    /**
     * Watch the returned player state object.
     */
    playerStateResponse: function() {
      this.handleNowPlaying()
    },
    
    /**
     * Watch the responses object.
     */
    responses: function() {
      this.handleNowPlaying()
    },

    /**
     * Watch our locally stored track data.
     */
    playerData: function() {
      this.$emit('spotifyTrackUpdated', this.playerData)

      this.$nextTick(() => {
        this.getAlbumColours()
      })
    }
  }
}
</script>

<style src="@/styles/components/now-playing.scss" lang="scss" scoped></style>
