<script>
import Icon from "fa-svelte";
import { faPlay } from "@fortawesome/free-solid-svg-icons/faPlay";
import { faPause } from "@fortawesome/free-solid-svg-icons/faPause";
import { faCog as faSettings } from "@fortawesome/free-solid-svg-icons/faCog";
import { faHome } from "@fortawesome/free-solid-svg-icons/faHome";
import { faCloudDownloadAlt as faDownload } from "@fortawesome/free-solid-svg-icons/faCloudDownloadAlt";
import { faPhotoVideo as faImageVideo} from "@fortawesome/free-solid-svg-icons/faPhotoVideo";
import { faImage } from "@fortawesome/free-solid-svg-icons/faImage";
import { faVideo } from "@fortawesome/free-solid-svg-icons/faVideo";
import { faStar as faFav } from "@fortawesome/free-solid-svg-icons/faStar";
import { faStar as faUnFav } from "@fortawesome/free-regular-svg-icons/faStar";
import { faSearch } from "@fortawesome/free-solid-svg-icons/faSearch";
import { faTimes as faClose } from "@fortawesome/free-solid-svg-icons/faTimes";
import { faSync } from "@fortawesome/free-solid-svg-icons/faSync";
import { faExclamationTriangle as faLoadError } from "@fortawesome/free-solid-svg-icons/faExclamationTriangle";
import { faSpinner } from "@fortawesome/free-solid-svg-icons/faSpinner";
import { faPlusCircle } from "@fortawesome/free-solid-svg-icons/faPlusCircle";
import { faMinusCircle } from "@fortawesome/free-solid-svg-icons/faMinusCircle";
import { faEye as faShow } from "@fortawesome/free-solid-svg-icons/faEye";
import { faEyeSlash as faHide } from "@fortawesome/free-solid-svg-icons/faEyeSlash";

import { faMobileAlt as faPortrait } from "@fortawesome/free-solid-svg-icons/faMobileAlt";
import { faDesktop as faLandscape } from "@fortawesome/free-solid-svg-icons/faDesktop";

import Settings from './Settings.svelte'

import { onMount, tick } from "svelte";
import { goto as ahref } from "@sapper/app";

import { get_posts, queryp } from "../_utils";

import { autoplay, autoplayinterval, imageVideo, portraitLandscape, favorite, over18, multireddit, prefetch, hires, oldreddit } from "../_prefs";
autoplay.useLocalStorage(true);
autoplayinterval.useLocalStorage(3);
imageVideo.useLocalStorage(0);
portraitLandscape.useLocalStorage(0);
favorite.useLocalStorage({});
over18.useLocalStorage(1);
multireddit.useLocalStorage({});
prefetch.useLocalStorage(true);
hires.useLocalStorage(false);
oldreddit.useLocalStorage(false);

export let params, slugstr;
export let posts;
export let after;
export let res;

let data;
let displayposts = [];
let uiVisible = true;
let numFavorite;
let tinygoto;
let title
let albumindex = 0

  $ : {
    if (currpost.is_album) {
      title = `(${albumindex + 1}/${currpost.preview.img.album.length}) ${currpost.title}`
    }
    else {
      title = currpost.title
    }

  }

let subreddit

$ : subreddit = slugstr ? slugstr.split('/')[1] : ""

let ismultireddit

$ : {
  // This tends to run on the server, where there is no localstorage
  if ($multireddit) {
  ismultireddit = $multireddit[currpost.subreddit]
  multiredditstr = ismultireddit ? "Remove from multi (m)" : "Add to multi (m)"
  }

  }

$: {
  if (gotoElWidth > 1000) {
    // padding on both sides
    let numGotoControlsInOneRow = (gotoElWidth - 154 * 2) / 32;
    let numGotoControlsRows =
      (displayposts.length + 5) / numGotoControlsInOneRow;
    tinygoto = numGotoControlsRows > 3;
  } else if (gotoElWidth > 800) {
    // padding on right side
    let numGotoControlsInOneRow = (gotoElWidth - (154 + 14)) / 32;
    let numGotoControlsRows =
      (displayposts.length + 5) / numGotoControlsInOneRow;
    tinygoto = numGotoControlsRows > 3;
  } else {
    // no padding
    let numGotoControlsInOneRow = (gotoElWidth - (14 + 14)) / 32;
    let numGotoControlsRows =
      (displayposts.length + 5) / numGotoControlsInOneRow;
    tinygoto = numGotoControlsRows > 3;
  }
}

let gotoElWidth;

$: loadError = res && !res.res.ok;
let loading = false;
let reloadstr = "Load more";
let navigation = false;

let imageVideoStr = "";
let portraitLandscapeStr = "";
let downloadstr = "";
let autoplaystr = "";
let over18str = "";
let deepsearchstr = "";
let multiredditstr = "";
let showhidestr = "Hide (h)";

let autoplaytimer;

let filterRef;
let filterExpanded = false;
let filterValue = "";

let showSettings = false;

let currpost = { title: "Loading .." };
let nexturls = [];

let index = 0;

async function loadMore() {
  if (!after) return;

  loading = true;
  reloadstr = "Loading ..";

  let newposts;

  ({ posts: newposts, after, ...res } = await get_posts(
    `https://reddit.com/${slugstr}.json?after=${after}&${queryp(params)}`
  ));

  // load `favorite` from localstorage
  for (let p of newposts) {
    p["favorite"] = !!$favorite[p.url];
  }

  // Combine `posts` and `newposts` and remove duplicates from multiple network requests
  posts = [...posts, ...newposts].reduce(
    (r, i) => (!r.some(j => i.id === j.id) ? [...r, i] : r),
    []
  );

  console.log("Total loaded: ", posts.length)

  loading = false;
  reloadstr = "Load more";
}

onMount(async () => {
  // Start autoplay by default
  if ($autoplay) {
    startAutoPlay();
  }
});

function startAutoPlay() {
  //console.log('START')
  autoplaytimer = setInterval(() => {
    // If `autoplay` is off and it is a video, the video will progress by itself via on:ended
    if ($autoplay && currpost.is_image) {
      //console.log('---- iNEXT')
      next();
    } else if (!$autoplay && currpost.is_video) {
      //console.log('---- vNEXT')
      next();
    }
  }, $autoplayinterval * 1000);

  autoplay.set(true);
}

function stopAutoPlay() {
  //console.log('STOP')
  clearInterval(autoplaytimer);
  autoplay.set(false);
}

function stopAndStartAutoPlay() {
  stopAutoPlay();

  startAutoPlay();
}

function toggleAutoPlay() {
  if ($autoplay) {
    stopAutoPlay();
  } else {
    startAutoPlay();
  }
}

function toggleImageVideo() {

  $imageVideo = $imageVideo + 1

  if ($imageVideo == 3) {
    $imageVideo = 0
  }
}

function togglePortraitLandscape() {
  $portraitLandscape = $portraitLandscape + 1

  if ($portraitLandscape == 3) {
    $portraitLandscape = 0
  }
}

let renderVideo = true;

// Some operations like fav/unfav causes video to re-render
// since we're changing post.favorite. This should help skip it
let skipRenderVideo = false;

$: {
  if (!skipRenderVideo) reMountVideo(currpost.preview);
  skipRenderVideo = false;
}

function reMountVideo() {
  renderVideo = false;
  setTimeout(() => (renderVideo = true), 0);
}

$: {
  numFavorite = displayposts.filter(item => item.favorite == true).length;

  if (!numFavorite) {
    downloadstr = `Nothing to download`;
  } else if (numFavorite == 1) {
    downloadstr = `Download ${numFavorite} file`;
  } else {
    downloadstr = `Download ${numFavorite} files`;
  }
  autoplaystr = `Autoplay is ${$autoplay ? "on" : "off"}`;
  deepsearchstr = `Search for ${filterValue}`;

  if ($over18 == 0) {
    over18str = "nsfw off"
  } else if ($over18 == 1) {
    over18str = "nsfw on"
  } else if ($over18 == 2) {
    over18str = "nsfw only"
  }

  if ($imageVideo == 0) {
    imageVideoStr = "Show both image and video"
  }
  else if ($imageVideo == 1) {
    imageVideoStr = "Show videos only"
  }
  else if ($imageVideo == 2) {
    imageVideoStr = "Show images only"
  }

  if ($portraitLandscape == 0) {
    portraitLandscapeStr = "Show all media"
  }
  else if ($portraitLandscape == 1) {
    portraitLandscapeStr = "Show only portrait media"
  }
  else if ($portraitLandscape == 2) {
    portraitLandscapeStr = "Show only landscape media"
  }

}

$: {
  if (displayposts[index]) {
    currpost = JSON.parse(JSON.stringify(displayposts[index]));

    nexturls = displayposts.slice(index + 1, index + 4);
  } else if (filterValue) {
    // We're here because user filtered the list

    // Unfortunately the filtered list is smaller than the current index
    // set index to last item
    if (displayposts.length > 0) {
      console.log("setting index from ", index, " to ", displayposts.length);
      index = displayposts.length - 1;
      console.log("loading more ..");
      loadMore();
    } else {
      // nothing was filtered
      index = 0;
      currpost = { title: "Nothing to show. Try changing filters or tweak/update URL." };
    }
  } else {
    if (res && res.res.ok) {
      // No media found
      currpost = { title: "Nothing to show. Try changing filters or tweak/update URL." };
    } else if (res && !res.res.ok) {
      // Invalid subreddit
      currpost = { title: "Error" };
    } else {
      // Default
      currpost = { title: "Loading .." };
    }

    nexturls = [];
  }
}

$: {
  let tmp = [];

  if ($over18 == 0) {
    tmp = posts.filter(item => item.over18 == false);
  }
  else if ($over18 == 1) {
    tmp = posts
  }
  else if ($over18 == 2) {
    tmp = posts.filter(item => item.over18 == true);
  }

  if (filterValue) {
    skipRenderVideo = true;
    tmp = tmp.filter(item =>
      item.title.toLowerCase().includes(filterValue.toLowerCase())
    );
  }

  // Filter only videos
  if ($imageVideo == 1) {
    tmp = tmp.filter(item => item.is_video);
  }
  // Filter only images
  else if ($imageVideo == 2) {
    tmp = tmp.filter(item => item.is_image);
  }

  if ($portraitLandscape == 1) {
    tmp = tmp.filter(item => (item.dims.width / item.dims.height) <= 1.2)
  }
  else if ($portraitLandscape == 2) {
    tmp = tmp.filter(item => (item.dims.width / item.dims.height) > 1.2)
  }

  displayposts = tmp;
}

function goto(i) {
  albumindex = 0
  index = i;

  let itemNum = displayposts.length - index;

  // If autoplay is on and we're jumping to 2 or 3, we must load
  if ((itemNum == 2 || itemNum == 3) && $autoplay) {
    console.log("[goto-to-2/3]: loading ..");
    loadMore();
  }

  // Last item
  if (itemNum === 1) {
    console.log("[goto-to-lastitem]: loading ..");
    loadMore();
  }

  if ($autoplay) stopAndStartAutoPlay();
}

function videoended() {
  next();
}

function next() {
  albumindex = 0
  let itemNum = displayposts.length - 1 - index;

  // Last item, dont go past the last item
  if (itemNum <= 1) {
    index = displayposts.length - 1;

    console.log("[lastitem, autoplay+filter?]: loading more ..");
    // Reached last item, possibly by autoplay + filter
    loadMore();

    return;
  }

  // Auto trigger on the last 4th item
  if (itemNum === 4 || itemNum === 3) {
    console.log("[4th last item, normal]: loading more ..");
    loadMore();
  }

  // If we're at 3rd/2nd last item with a filter, the user
  // possibly just filtered the list and ended up here.
  // trigger a load more. We dont want to do it always since
  // we normally trigger loadmore @3rd last item. Always doing it
  // Would end up with 2 requests to reddit.com
  if (itemNum === 2 && filterValue) {
    console.log("[2nd last item, filtering?]: loading more ..");
    loadMore();
  }

  index += 1;

  if ($autoplay) stopAndStartAutoPlay();
}

function prev() {
  albumindex = 0

  if (index === 0) return;
  index -= 1;

  if (displayposts.length - index === 3) {
    loadMore();
  }
  if ($autoplay) stopAndStartAutoPlay();
}

function toggleUIVisiblity() {
  uiVisible = !uiVisible;

  showhidestr = uiVisible ? "Hide (h)" : "Show (h)"
}

function toggleSettings() {
  showSettings = !showSettings;
}

function gotoDeepSearch() {

  let prefix = "";
  if (slugstr) {
    prefix = `/r/${subreddit}`;
  } else {
    prefix = ``;
  }

  navigation = true;
  ahref(
    `${prefix}/search?q=${filterValue}&restrict_sr=on&include_over_18=on&sort=relevant&t=all`
  );
}

function hideSettings() {
  showSettings = false;
}

async function expandFilter() {
  filterExpanded = true;

  await tick();
  // Focus the input if we just opened it
  if (filterExpanded) filterRef.querySelector("input").focus();
}

async function toggleFilter() {
  filterExpanded = !filterExpanded;

  await tick();
  // Focus the input if we just opened it
  if (filterExpanded) filterRef.querySelector("input").focus();
}

async function downloadFiles() {
  window.open("/download", "_blank");
}

function openMedia() {
  window.open(currpost.url, "_blank");
}

function toggleMultireddit() {

  if ($multireddit.hasOwnProperty(currpost.subreddit)) {

    $multireddit[currpost.subreddit] = undefined;
    $multireddit = JSON.parse(JSON.stringify($multireddit));


  }
  else {
    $multireddit[currpost.subreddit] = {
      preview: currpost.preview.img.default
    }
  }
    multireddit.set($multireddit)
}

function openSubReddit() {
  if (slugstr != currpost.subredditp) {
    ahref(`/r/${currpost.subreddit}`);
  }
}

function openComments() {
  window.open(`https://reddit.com/${currpost.permalink}`, "_blank");
}

function openCommentsOld() {
  window.open(`https://old.reddit.com/${currpost.permalink}`, "_blank");
}

function openDuplicates() {
  window.open(`https://old.reddit.com/${currpost.subredditp}/duplicates/${currpost.id}`, "_blank");
}


function toggleOver18() {
  $over18 = $over18 + 1

  if ($over18 == 3) {
    $over18 = 0
  }
  over18.set($over18);
}

function removeAllFavorite(removeAllFromLocalStorage) {
  skipRenderVideo = true;

  for (const [i, post] of displayposts.entries()) {
    // For reactivity
    displayposts[i].favorite = false;

    // If removeAllFromLocalStorage is true, then we'll remove everythign in one shot
    // no need to do it one by one
    if (removeAllFromLocalStorage == false) {
      // Localstorage
      $favorite[post.url] = undefined;
      $favorite = JSON.parse(JSON.stringify($favorite));

      favorite.set($favorite);
    }
  }

  if (removeAllFromLocalStorage) {
    favorite.set({});
  }
}
function toggleFavorite() {
  skipRenderVideo = true;
  displayposts[index].favorite = !displayposts[index].favorite;

  let url = displayposts[index].url;
  if (displayposts[index].favorite) {
    // Set into localStorage
    $favorite[url] = displayposts[index];
    favorite.set($favorite);
  } else {
    // setting a value in js which after JSON.parse(JSON.stringify(d)) removes it

    $favorite[url] = undefined;
    $favorite = JSON.parse(JSON.stringify($favorite));

    favorite.set($favorite);
  }
}

function albumPrev() {
  if (!currpost.is_album) return

  if (albumindex == 0) {
    albumindex = currpost.preview.img.album.length - 1
  }
  else {
    albumindex -= 1
  }
  if ($autoplay) stopAndStartAutoPlay();

}

function albumNext() {
  if (!currpost.is_album) return

  if (albumindex == (currpost.preview.img.album.length -1)) {
    albumindex = 0
  }
  else {
    albumindex += 1
  }

  if ($autoplay) stopAndStartAutoPlay();

}

function keydown(event) {

  // up
  if (event.keyCode == 38) {
    albumPrev()
  }

  if (event.keyCode == 40) {
    albumNext()
  }

  // m
  if (event.keyCode == 77) {
    toggleMultireddit()
  }

  // q, p
  if (event.keyCode == 81 || event.keyCode == 80) {
    toggleAutoPlay();
  }

  // slash, f
  if (event.keyCode == 191 || event.keyCode == 70) {
    expandFilter();
    // We need this, otherwise filter box will have '/' because of autofocus
    event.preventDefault();
  }

  // x
  if (event.keyCode == 88) {
    if (event.shiftKey) {
      removeAllFavorite(event.ctrlKey); // if ctrl+shift+x is remove everything from localstorage
    } else {
      toggleFavorite();
    }
  }

  if (event.ctrlKey) {
    return;
  }

  // r
  if (event.keyCode == 82) {
    if ($oldreddit) {
      openCommentsOld();
    }
    else {
      openComments();
    }
  }

  // o
  if (event.keyCode == 79) {
    openCommentsOld();
  }

  // i
  if (event.keyCode == 73) {
    openMedia();
  }

  // l
  if (event.keyCode == 76) {
    openDuplicates();
  }

  // h
  if (event.keyCode == 72) {
    toggleUIVisiblity();
  }

  // Left Arrow, a, k, Page-up
  if (event.keyCode == 37 || event.keyCode == 65 || event.keyCode == 75 || event.keyCode == 33) {
    prev();
  }
  // Right Arrow, d, j, Space, Page-down
  else if (
    event.keyCode == 39 ||
    event.keyCode == 68 ||
    event.keyCode == 74 ||
    event.keyCode == 32 ||
    event.keyCode == 34
  ) {
    next();
  }
}
</script>

<style lang="sass">
@mixin hover()
  @media not all and (pointer:coarse)
    &:hover
      @content

$yellow: #f9ab00

$text-color: #fafafa
$accent-color: white
$favorite-color: #fbbc04
$favorite-border-color: #e37400
$over18-color: #ea4335
$over18-border-color: #ea4335
$ismulti-color: #ea4335
$isnotmulti-color: #34a853

.hide
  display: none !important

.wrapper
  height: 100vh

  display: grid
  justify-items: center
  align-items: center

  .hero
    height: 100vh
    width: 100%

    grid-template-columns: auto
    display: grid
    justify-items: center
    align-items: center

    .settings
      z-index: 10
      position: absolute
      top: 0
      right: 0
      color: $text-color
      font-size: 1rem
      padding: 1.5rem 2rem


      .btn
        user-select: none
        cursor: pointer
        color: rgba(white, 80%)
        margin-left: 10px
        font-size: 1.2rem

        &.cog
          font-size: 1.1rem

        &.showhide
          top: 1px

        &.showSettings
          color: white

        @include hover()
          color: white

    .title
      z-index: 10
      position: absolute
      top: 0
      background-color: rgba(0, 0, 0, 0.4)
      color: $text-color
      font-size: 1.5rem
      max-width: 77%
      padding: 1rem
      border-radius: 3px

      .subreddit
        font-size: 1rem
        cursor: pointer
        color: darken($text-color, 30%)
        width: fit-content
        user-select: none

        &.ismulti
          color: $isnotmulti-color

        @include hover()
          color: $text-color

        .subredditwrapper
          display: inline-block
          margin-left: 7px
          top: 3px
          position: relative
          opacity: 0.5

          &.ismulti
            color: darken($text-color, 30%)

            @include hover()
              color: $ismulti-color
              opacity: 1

          @include hover()
            color: $isnotmulti-color
            opacity: 1

      .fav
        user-select: none
        cursor: pointer
        top: 2px
        position: relative
        margin-right: 12px

        @include hover()
          color: $favorite-color

      &.favorite
        color: $favorite-color

    .goto
      user-select: none
      z-index: 5
      position: absolute
      background-color: rgba(0, 0, 0, 0.6)
      bottom: 0
      display: grid
      grid-row-gap: 5px
      padding: 1rem 11rem
      border-radius: 3px
      color: $text-color
      width: 100%
      grid-template-columns: repeat(auto-fill, minmax(32px, 1fr))

      &.tinygoto
        grid-template-rows: auto 1fr
        grid-template-columns: 1fr

        .btnwrapper
          grid-template-columns: repeat(auto-fill, minmax(32px, 1fr))
          display: grid

        .numswrapper
          grid-template-columns: repeat(auto-fit, minmax(1rem, auto))
          grid-auto-columns: max-content
          display: grid
          grid-gap: 0.2rem

          .nums
            border-bottom: 3px solid rgba(white, 30%)
            height: 1rem
            cursor: pointer

            @include hover()
              border-bottom: 3px solid $accent-color !important

            &.currnum
              border-bottom: 3px solid $accent-color !important

              &.album
                border-bottom: 3px dotted $accent-color !important

            &.favorite
              border-bottom: 3px solid $favorite-color

            &.over18
              border-bottom: 3px solid $over18-color

          p
            display: none

          .reload
            grid-column: span 2

      .btnwrapper, .numswrapper
        display: contents

      .btnwrapper
        .reload
          bottom: -1px

      .btn
        text-align: center
        padding-top: 2px
        color: rgba(white, 30%)

        &.reload
          cursor: pointer

          &.loaderror
            color: $over18-color

          @include hover()
            color: white

        &.deepsearch
          grid-column: span 4
          bottom: 2px
          cursor: pointer
          justify-self: center

          &:hover p
            color: $accent-color
            border: 1px solid $accent-color

          p
            margin: 0
            font-size: 0.9rem
            color: darken($accent-color, 30%)
            border: 1px solid darken($accent-color, 30%)
            border-radius: 3px
            padding: 0 1rem

        &.over18wrapper

          cursor: pointer
          grid-column: span 2
          justify-self: center

          &.over18

            p
              border: 1px solid rgba(white, 30%)
              color: rgba(white, 30%)

              @include hover()
                border: 1px solid rgba(white, 60%)
                color: rgba(white, 60%)

          p
            font-size: 0.9rem
            border: 1px solid $over18-color
            border-radius: 3px
            color: $over18-color
            margin: 0
            //margin-right: 13px
            width: 58px
            font-family: "Roboto Condensed", sans-serif
            position: relative
            top: -1px

        &.imagevideo
          cursor: pointer
          font-size: 1.4rem
          bottom: 2px
          color: white

        &.portraitlandscape
          cursor: pointer
          font-size: 1.4rem
          bottom: 2px
          color: white

          & :global(.landscape)
            transform: rotate(270deg)

        &.download
          cursor: default
          font-size: 1.4rem
          bottom: 2px

          &.dlready
            color: rgba($favorite-color, 90%)
            cursor: pointer

            @include hover()
              color: $yellow

        &.playpause
          cursor: pointer
          top: 1px

          // When it is play icon, make it white
          &.play
            color: white

          @include hover()
            color: white

        &.filter
          cursor: pointer
          top: 1px

          @include hover()
            color: white

          &.filter.filterExpanded
            grid-column: span 3
            top: -2px

            input
              width: 90%
              margin-left: 9px
              padding-left: 5px
              padding-right: 5px
              border: 1px solid rgba(white, 60%)
              background-color: rgba(0, 0, 0, 0)
              color: white

      span
        position: relative

        &.favorite

          p.small
            //background-color: $favorite-color
            border-bottom: 3px solid $favorite-border-color !important
            color: $favorite-color

          img.small
            border-color: $favorite-border-color !important

        &.over18

          p.small
            color: $over18-color
            //background-color: $over18-color
            border-bottom: 3px solid $over18-border-color

          img.small
            border-color: $over18-border-color

        p.small
          margin: 0 2px
          text-align: center
          cursor: pointer
          border-bottom: 3px solid rgba(0, 0, 0, 0)

          &.curr
            background-color: rgba(255, 255, 255, 0.2)
            border-bottom: 3px solid $accent-color !important

            &.album
              border-bottom: 3px dashed $accent-color !important

        img.small
          z-index: 10
          width: 0px
          height: 0px
          opacity: 0
          position: absolute
          bottom: 35px
          border: 2px solid white
          background-color: lighten(black, 30%)
          min-width: 100px
          pointer-events: none
          object-fit: cover

        &:hover p.small
          background-color: rgba(255, 255, 255, 0.1)
          border-bottom: 3px solid $accent-color !important

        &:hover img
          width: auto
          height: 100px
          opacity: 1

    .control
      position: absolute
      height: 100%
      z-index: 5

      &.prev
        left: 0
        width: 15%
        //background-color: rgba(255, 138, 138, 0.38)

        @include hover()
          background: linear-gradient(90deg, rgba(0,0,0,0.3) 0%, rgba(255,255,255,0) 100%)

      &.next
        right: 0
        width: 85%
        //background-color: rgba(138, 255, 233, 0.38)

    .image
      height: 100%
      width: 100%
      background-size: contain
      background-repeat: no-repeat
      background-position: center

    .video
      height: 100%
      width: 100%

  .prefetch
    display: none

  @media (max-width: 1000px)
    .hero
      .goto
        padding: 1rem 11rem 1rem 1rem

  @media (max-width: 800px)
    .hero
      .goto
        padding: 1rem
        //grid-template-columns: 32px 32px 32px 32px 32px repeat(auto-fit, minmax(32px, 1fr))
        //grid-template-rows: 1fr 1fr

        &.tinygoto

          .nums
            height: 0.1rem !important
        img.small
          display: none

.tooltip
  position: relative
  z-index: 2
  cursor: pointer

.ttbefore
  position: absolute
  bottom: 120%
  left: 50%
  margin-bottom: 5px
  margin-left: -30px
  padding: 5px 4px
  width: max-content
  border-radius: 3px
  background-color: black
  color: #fff

  background-color: rgba(white, 90%)
  color: black

  content: attr(data-tooltip)
  text-align: center
  font-size: 0.8rem
  line-height: 1.2

.ttafter
  position: absolute
  bottom: 120%
  left: 50%
  margin-left: -5px
  width: 0
  border-top: 5px solid rgba(white, 90%)
  border-right: 5px solid transparent
  border-left: 5px solid transparent
  content: " "
  font-size: 0
  line-height: 0

.tooltip
  &:before, &:after
    visibility: hidden
    opacity: 0
    pointer-events: none

  &:before
    @extend .ttbefore

  &.bottom:before
    @extend .ttbefore
    bottom: -170%

  &:after
    @extend .ttafter

  &.bottom:after
    @extend .ttafter
    bottom: -40%
    border-bottom: 5px solid rgba(white, 90%)
    border-top: 5px solid transparent

  &:hover
    &:before, &:after
      visibility: visible
      opacity: 1

</style>

<svelte:window on:keydown={keydown} />
<svelte:head>
  <title>redditpx - {slugstr ? `r/${subreddit}` : 'reddit.com'}</title>
</svelte:head>

<template lang="pug">
.wrapper
  .hero
    .control.prev(on:click="{prev}")
    .title(class:hide="{uiVisible == false}", class:favorite="{currpost.favorite}")
      +if('displayposts.length')
        span.fav(on:click|stopPropagation|preventDefault="{toggleFavorite}")
          Icon(icon="{currpost.favorite ? faFav : faUnFav}")
      +if('currpost.dims')
        | {title} ({currpost.dims.width}x{currpost.dims.height})
        +else
          | {title}
      +if('currpost.subreddit')
        .subreddit(on:click='{openSubReddit}', class:ismulti='{ismultireddit}') {currpost.subredditp}
          .subredditwrapper.tooltip.bottom(data-tooltip='{multiredditstr}', on:click|stopPropagation='{toggleMultireddit}', class:ismulti='{ismultireddit}')
            Icon(icon="{ismultireddit ? faMinusCircle : faPlusCircle}")
    .settings
      a.home(rel="prefetch", href="/home", class:hide='{uiVisible == false}')
        span.btn.tooltip.bottom(data-tooltip="Home")
          Icon(icon="{faHome}")
      span.btn.cog(on:click='{toggleSettings}', class:showSettings='{showSettings}', class:hide='{uiVisible == false}')
        Icon(icon="{faSettings}")
      span.btn.tooltip.bottom.showhide(data-tooltip="{showhidestr}", on:click="{toggleUIVisiblity}")
        Icon(icon="{uiVisible ? faHide : faShow }")
      .div(class:hide='{uiVisible == false}')
        Settings(bind:showSettings)
    +if('currpost.is_image && !currpost.is_album')
      +if('$hires')
        .image(style="background-image: url('{currpost.url}')")
        +else()
          .image(style="background-image: url('{currpost.preview.img.default}')")
      +elseif('currpost.is_video && renderVideo')
        video.video(autoplay, loop='{!$autoplay}', playsinline, muted, on:ended="{videoended}")
          +if('currpost.preview.vid.webm')
            source(src="{currpost.preview.vid.webm}")
          +if('currpost.preview.vid.mp4')
            source(src="{currpost.preview.vid.mp4}")
      +elseif('currpost.is_album')
        +if('currpost.preview.img.album[albumindex].is_video')
          video.video(autoplay, loop='{!$autoplay}', playsinline, muted, on:ended="{videoended}")
            source(src="{currpost.preview.img.album[albumindex].hires}")
          +else()
            +if('$hires')
              .image(style="background-image: url('{currpost.preview.img.album[albumindex].hires}')")
              +else()
                // Use reddit's optimized image for index 0
                +if('albumindex === 0')
                  .image(style="background-image: url('{currpost.preview.img.default}')")
                  +else()
                    .image(style="background-image: url('{currpost.preview.img.album[albumindex].default}')")
    .control.next(on:click="{next}")
    +if('displayposts.length || posts.length')
      .goto(class:tinygoto='{tinygoto}', class:hide="{uiVisible == false}", bind:clientWidth='{gotoElWidth}')
        .btnwrapper
          span.btn.playpause.tooltip(
            data-tooltip="{autoplaystr}",
            class:play="{$autoplay}",
            on:click="{toggleAutoPlay}"
          )
            Icon(icon="{$autoplay ? faPause : faPlay}")
          span.btn.download.tooltip(
            on:click="{downloadFiles}",
            data-tooltip="{downloadstr}",
            class:dlready="{numFavorite}"
          )
            Icon(icon="{faDownload}")
          span.btn.portraitlandscape.tooltip(
            on:click="{togglePortraitLandscape}",
            data-tooltip="{portraitLandscapeStr}",
            class:active="{$portraitLandscape}"
          )
            +if('$portraitLandscape == 0')
              Icon(icon="{faLandscape}")
              +elseif('$portraitLandscape == 1')
                Icon(icon="{faPortrait}")
              +elseif('$portraitLandscape == 2')
                Icon(class="landscape", icon="{faPortrait}")
          span.btn.imagevideo.tooltip(
            data-tooltip="{imageVideoStr}",
            on:click="{toggleImageVideo}"
          )
            +if('$imageVideo == 0')
              Icon(icon="{faImageVideo}")
              +elseif('$imageVideo == 1')
                Icon(icon="{faVideo}")
              +elseif('$imageVideo == 2')
                Icon(icon="{faImage}")
          +if('tinygoto')
            span.btn.reload.tooltip(data-tooltip="{reloadstr}", on:click='{loadMore}', class:loaderror='{loadError}')
              +if('loading')
                Icon(icon="{faSpinner}")
                +else()
                  Icon(icon="{faSync}")
          span.btn.filter.tooltip(
            class:filterExpanded="{filterExpanded}",
            on:click="{toggleFilter}",
            data-tooltip="Filter ( / )",
            bind:this='{filterRef}'
            class:dlready="{numFavorite}"
          )
            +if('filterExpanded')
              input(bind:value='{filterValue}', on:click|stopPropagation, on:keydown|stopPropagation, type="text")
              +else
                Icon(icon="{faSearch}")
          span.btn.over18wrapper.tooltip(
            data-tooltip="{over18str}",
            class:over18="{!$over18}",
            on:click="{toggleOver18}"
          )
            p {over18str}
        .numswrapper
          +each('displayposts as post, i')
            span.nums(
              class:currnum="{index === i}",
              class:album="{currpost.is_album}",
              class:favorite="{displayposts[i].favorite}",
           e  class:over18="{displayposts[i].over18}",
              on:click="{function(){goto(i)}}"
            )
              //img.small(alt="{displayposts[i].title}", src="{displayposts[i].preview.img.default}")
              img.small(alt="{displayposts[i].title}", src="{displayposts[i].thumbnail}")
              p.small(class:curr="{index === i}", class:album="{currpost.is_album}") {i+1}
          +if('filterValue')
            span.btn.deepsearch.tooltip(data-tooltip="{deepsearchstr}", on:click='{gotoDeepSearch}')
              p deep search 🡒
          +if('!tinygoto')
            span.btn.reload.tooltip(data-tooltip="{reloadstr}", on:click='{loadMore}', class:loaderror='{loadError}')
              +if('loading')
                Icon(icon="{faSpinner}")
                +else()
                  Icon(icon="{faSync}")
  +if('$prefetch')
    .prefetch
      +each('nexturls as nexturl')
        +if('$hires')
          img(alt="prefetch", src="{nexturl.url}")
          +else()
            img(alt="prefetch", src="{nexturl.preview.img.default}")
        +if('nexturl.is_video')
          video
            +if('nexturl.preview.vid.webm')
              source(src="{nexturl.preview.vid.webm}")
            +if('nexturl.preview.vid.mp4')
              source(src="{nexturl.preview.vid.mp4}")
</template>
