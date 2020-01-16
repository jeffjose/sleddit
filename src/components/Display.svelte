<script>
import Icon from "fa-svelte";
import { faPlay } from "@fortawesome/free-solid-svg-icons/faPlay";
import { faPause } from "@fortawesome/free-solid-svg-icons/faPause";
import { faCog as faSettings } from "@fortawesome/free-solid-svg-icons/faCog";
import { faCloudDownloadAlt as faDownload } from "@fortawesome/free-solid-svg-icons/faCloudDownloadAlt";
import { faStar as faFav } from "@fortawesome/free-solid-svg-icons/faStar";
import { faStar as faUnFav } from "@fortawesome/free-regular-svg-icons/faStar";
import { faSearch } from "@fortawesome/free-solid-svg-icons/faSearch";
import { faTimes as faClose } from "@fortawesome/free-solid-svg-icons/faTimes";

import { onMount, tick } from "svelte";
  import {goto as ahref} from "@sapper/app"

import { get_posts } from "../_utils";

import { autoplay, selected, over18 } from "../_prefs";
autoplay.useLocalStorage(true);
selected.useLocalStorage({});
over18.useLocalStorage(true);

export let params, slugstr;

let data;
let posts = [];
let displayposts = [];
let res;
let after;
let uiVisible = true;
let numSelected;
let tinygoto;

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

let urlParams = Object.entries(params)
  .map(([key, val]) => `${key}=${val}`)
  .join("&");

//let gotoEl
//$: {
//    let gotoElWidth = gotoEl.clientWidth - (getComputedStyle(gotoEl).paddingLeft + getComputedStyle(gotoEl).paddingRight)
//    let gotoControls = gotoElWidth / 32
//    numPostsOver50 = (gotoControls / displayposts.length ) > 2
//    console.log('width: ', gotoElWidth, 'controls: ', gotoControls, 'posts ', displayposts.length, 'rows: ', gotoControls / displayposts.length)
//  }

let gotoElWidth;

let downloadstr = "";
let autoplaystr = "";
let over18str = "";

let autoplayinterval = 3;
let autoplaytimer;

let filterRef;
let filterExpanded = false;
let filterValue = "";

let showSettings = false;

let currpost = { title: "Loading .." };
let nexturls = [];

let index = 0;

let errorinputValue = slugstr;

async function loadMore() {
  if (!after) return;

  let newposts;

  ({ posts: newposts, after, ...res } = await get_posts(
    `https://reddit.com/${slugstr}.json?after=${after}&${urlParams}`
  ));

  // load `selected` from localstorage
  for (let p of posts) {
    p["selected"] = !!$selected[p.url];
  }

  // Combine `posts` and `newposts` and remove duplicates from multiple network requests
  posts = [...posts, ...newposts].reduce(
    (r, i) => (!r.some(j => i.id === j.id) ? [...r, i] : r),
    []
  );
}

onMount(async () => {
  ({ posts, after, ...res } = await get_posts(
    `https://reddit.com/${slugstr}.json?${urlParams}`
  ));

  // Load `selected` from localstorage
  for (let p of posts) {
    p["selected"] = !!$selected[p.url];
  }

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
  }, autoplayinterval * 1000);

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

let renderVideo = true;

// Some operations like fav/unfav causes video to re-render
// since we're changing post.selected. This should help skip it
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
  numSelected = displayposts.filter(item => item.selected == true).length;

  if (!numSelected) {
    downloadstr = `nothing to download`;
  } else if (numSelected == 1) {
    downloadstr = `download ${numSelected} file`;
  } else {
    downloadstr = `download ${numSelected} files`;
  }
  autoplaystr = `autoplay is ${$autoplay ? "on" : "off"}`;
  over18str = `nsfw is ${$over18 ? "on" : "off"}`;
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
      currpost = { title: "Nothing to show" };
    }
  } else {
    if (res && res.res.ok) {
      // No media found
      currpost = { title: "Nothing to show" };
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

  if (!$over18) {
    tmp = posts.filter(item => item.over18 == false);
  } else {
    tmp = posts;
  }

  if (filterValue) {
    skipRenderVideo = true;
    tmp = tmp.filter(item =>
      item.title.toLowerCase().includes(filterValue.toLowerCase())
    );
  }

  displayposts = tmp;
}

function goto(i) {
  console.log("goto");
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
  // Last item, dont go past the last item
  if (displayposts.length - index == 1) {
    index = displayposts.length - 1;

    console.log("[lastitem, autoplay+filter?]: loading more ..");
    // Reached last item, possibly by autoplay + filter
    loadMore();
    return;
  }

  index += 1;

  // Auto trigger on the last 3rd item
  if (displayposts.length - index === 3) {
    console.log("[3rd last item, normal]: loading more ..");
    loadMore();
  }

  // If we're at 2nd last item with a filter, the user
  // possibly just filtered the list and ended up here.
  // trigger a load more. We dont want to do it always since
  // we normally trigger loadmore @3rd last item. Always doing it
  // Would end up with 2 requests to reddit.com
  if (displayposts.length - index === 2 && filterValue) {
    console.log("[2nd last item, filtering?]: loading more ..");
    loadMore();
  }

  if ($autoplay) stopAndStartAutoPlay();
}

function prev() {
  if (index === 0) return;
  index -= 1;

  if (displayposts.length - index === 3) {
    loadMore();
  }
  if ($autoplay) stopAndStartAutoPlay();
}

function toggleUIVisiblity() {
  uiVisible = !uiVisible;
}

function toggleSettings() {
  showSettings = !showSettings;
}

function gotoDeepSearch() {
  ahref("/r/gifs")
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

function openSubReddit() {
  window.open(`https://reddit.com/${currpost.permalink}`, "_blank");
}

function openSubRedditOld() {
  window.open(`https://old.reddit.com/${currpost.permalink}`, "_blank");
}

function toggleOver18() {
  skipRenderVideo = true;
  over18.set(!$over18);
}

function removeAllSelected(removeAllFromLocalStorage) {
  skipRenderVideo = true;

  for (const [i, post] of displayposts.entries()) {
    // For reactivity
    displayposts[i].selected = false;

    // If removeAllFromLocalStorage is true, then we'll remove everythign in one shot
    // no need to do it one by one
    if (removeAllFromLocalStorage == false) {
      // Localstorage
      $selected[post.url] = undefined;
      $selected = JSON.parse(JSON.stringify($selected));

      selected.set($selected);
    }
  }

  if (removeAllFromLocalStorage) {
    selected.set({});
  }
}
function toggleSelected() {
  skipRenderVideo = true;
  displayposts[index].selected = !displayposts[index].selected;

  let url = displayposts[index].url;
  if (displayposts[index].selected) {
    // Set into localStorage
    $selected[url] = displayposts[index];
    selected.set($selected);
  } else {
    // setting a value in javascript which after JSON.parse(JSON.stringify(d)) removes it

    $selected[url] = undefined;
    $selected = JSON.parse(JSON.stringify($selected));

    selected.set($selected);
  }
}

function keydown(event) {
  console.log(event.keyCode);

  // q, p
  if (event.keyCode == 81 || event.keyCode == 80) {
    toggleAutoPlay();
  }

  // slash, f
  if (event.keyCode == 191 || event.keyCode == 70) {
    expandFilter();
    // We need this, otherwise filter box will have '/' because of autofocus
    console.log("preventDefault");
    event.preventDefault();
  }

  // x
  if (event.keyCode == 88) {
    if (event.shiftKey) {
      removeAllSelected(event.ctrlKey); // if ctrl+shift+x is remove everything from localstorage
    } else {
      toggleSelected();
    }
  }

  if (event.ctrlKey) {
    return;
  }

  // r
  if (event.keyCode == 82) {
    openSubReddit();
  }

  // o
  if (event.keyCode == 79) {
    openSubRedditOld();
  }

  // i
  if (event.keyCode == 73) {
    openMedia();
  }

  // h
  if (event.keyCode == 72) {
    toggleUIVisiblity();
  }

  // Left Arrow, a, k
  if (event.keyCode == 37 || event.keyCode == 65 || event.keyCode == 75) {
    prev();
  }
  // Right Arrow, d, j, Space
  else if (
    event.keyCode == 39 ||
    event.keyCode == 68 ||
    event.keyCode == 74 ||
    event.keyCode == 32
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
$selected-color: #fbbc04
$selected-border-color: #e37400
$over18-color: #ea4335
$over18-border-color: #ea4335

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

    .error
      z-index: 10

    .settings
      z-index: 10
      position: absolute
      top: 0
      right: 0
      color: $text-color
      font-size: 1rem
      padding: 1.5rem

      .btn
        user-select: none
        cursor: pointer
        color: rgba(white, 80%)

        &.showSettings
          color: white

        @include hover()
          color: white

      .settingspanel
        position: fixed
        background-color: black
        left: 25%
        top: 25%
        width: 50%
        height: 50%
        border-radius: 5px
        border: 1px solid white
        padding: 1rem
        display: none

        grid-template-rows: [head-start] 2.5rem [head-end contents-start] 2fr [contents-end]

        &.showSettings
          display: grid
          grid-gap: 1rem

        .contents
          grid-row: contents
          display: grid
          grid-template-columns: 1fr 2fr

          .nav
            font-size: 1.1rem

            display: grid
            grid-auto-rows: max-content
            grid-gap: 5px

            align-items: center
            justify-items: center

            // flow items one below other
            grid-auto-flow: row

            div
              padding: 0.5rem 1rem
              border-bottom: 3px solid rgba(0, 0, 0, 0)
              width: 100%
              height: 100%
              cursor: pointer

              @include hover()
                background-color: rgba(255, 255, 255, 0.2)
                border-bottom: 3px solid $accent-color

          .options
            background-color: rgba(black, 0%)
            border-left: 1px solid white

        .close
          position: absolute
          top: 1rem
          color: rgba(white, 60%)
          cursor: pointer
          right: 1rem

          @include hover()
            color: white

        .head
          font-size: 1.5rem
          align-self: center

          :global(svg)
            position: relative
            top: 3px
            margin-right: 10px

        @media (max-width: 1600px)
          width: 80%
          left: 10%

        @media (max-width: 1000px)
          width: 90%
          left: 5%

        @media (max-width: 800px)

          .contents
            grid-template-rows: 3rem auto
            grid-template-columns: unset

            .nav
              // flow items to the right
              grid-auto-flow: column
              grid-auto-rows: unset
              grid-auto-columns: max-content

            .options
              border-left-width: 0px
              border-top: 1px solid white

    .title
      z-index: 10
      position: absolute
      top: 0
      background-color: rgba(0, 0, 0, 0.4)
      color: $text-color
      font-size: 1.5rem
      max-width: 500px
      padding: 1rem
      border-radius: 3px

      .fav
        user-select: none
        cursor: pointer
        top: 2px
        position: relative
        margin-right: 12px

        @include hover()
          color: $selected-color

      &.selected
        color: $selected-color

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

            &.selected
              border-bottom: 3px solid $selected-color

            &.over18
              border-bottom: 3px solid $over18-color

          p
            display: none

      .btnwrapper, .numswrapper
        display: contents

      .btn
        text-align: center
        padding-top: 2px
        color: rgba(white, 30%)

        &.deepsearch
          grid-column: span 3
          bottom: 2px
          cursor: pointer

          p
            margin: 0
            font-size: 0.9rem
            color: $accent-color
            border: 1px solid $accent-color
            border-radius: 3px

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
            width: 35px
            font-family: "Roboto Condensed", sans-serif
            position: relative
            top: -1px

        &.download
          cursor: default
          font-size: 1.4rem
          bottom: 2px

          &.dlready
            color: rgba($selected-color, 90%)
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

        &.selected

          p.small
            //background-color: $selected-color
            border-bottom: 3px solid $selected-border-color !important
            color: $selected-color

          img.small
            border-color: $selected-border-color !important

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

.tooltip
  &:before, &:after
    visibility: hidden
    opacity: 0
    pointer-events: none

  &:before
    position: absolute
    bottom: 120%
    left: 50%
    margin-bottom: 5px
    margin-left: -30px
    padding: 5px 4px
    width: 60px
    border-radius: 3px
    background-color: #000
    background-color: hsla(0, 0%, 20%, 0.9)
    color: #fff
    content: attr(data-tooltip)
    text-align: center
    font-size: 0.8rem
    line-height: 1.2

  &:after
    position: absolute
    bottom: 120%
    left: 50%
    margin-left: -5px
    width: 0
    border-top: 5px solid #000
    border-top: 5px solid hsla(0, 0%, 20%, 0.9)
    border-right: 5px solid transparent
    border-left: 5px solid transparent
    content: " "
    font-size: 0
    line-height: 0

  &:hover
    &:before, &:after
      visibility: visible
      opacity: 1

</style>

<svelte:window on:keydown={keydown} />
<svelte:head>
  <title>redditpx - {slugstr ? slugstr : 'reddit.com'}</title>
</svelte:head>

<template lang="pug">
.wrapper
  .hero
    +if('res && !res.res.ok')
      .error
        input(bind:value='{errorinputValue}', on:keydown|stopPropagation)
        a(href="{'/r/' + errorinputValue}") click
        a(href="/r/gifs") gifs
    .control.prev(on:click="{prev}")
    .title(class:hide="{uiVisible == false}", class:selected="{currpost.selected}")
      +if('displayposts.length')
        span.fav(on:click|stopPropagation|preventDefault="{function(){toggleSelected()}}")
          Icon(icon="{currpost.selected ? faFav : faUnFav}")
      | {currpost.title}
      a(href='{currpost.subreddit}', rel="prefetch") {currpost.subreddit}
      a(href='/r/gifs', rel="prefetch") /r/gifs
      a(href='/r/comics', rel="prefetch") /r/comics
    .settings(class:hide="{uiVisible == false}")
      span.btn(on:click='{function() { toggleSettings()}}', class:showSettings='{showSettings}')
        Icon(icon="{faSettings}")
      .settingspanel(class:showSettings='{showSettings}')
        .head
          Icon(icon="{faSettings}")
          | Settings
        .close(on:click='{function(){ hideSettings()}}')
          Icon(icon="{faClose}")
        .contents
          .nav
            div General
            div Keybindings
          .options
            div
              p autoplay on/off
              p download files
              p nsfw on/off
              p autoplay time
              p make all fav
              p remove all fav
              p prefetch?
              p always show hd?
              p remove duplicates
            div
              p play/pause q,p
              p next space, right, d,  j
              p prev left, a, k
              p hide h
              p fav x
              p undofav shift-x
              p undoallfav ctrl-shift-x
              p undoallfav ctrl-shift-x
              p filter /, f
              p subreddit-old,  o
              p subreddit-default,  r
              p highres,  i
    +if('currpost.is_image')
      .image(style="background-image: url('{currpost.preview.img.default}')")
      +elseif('currpost.is_video && renderVideo')
        video.video(autoplay, loop='{!$autoplay}', playsinline, muted, on:ended="{videoended}")
          +if('currpost.preview.vid.webm')
            source(src="{currpost.preview.vid.webm}")
          +if('currpost.preview.vid.mp4')
            source(src="{currpost.preview.vid.mp4}")
          //img(alt="foo", src='{currpost.preview.vid.gif}')

    .control.next(on:click="{next}")
    +if('displayposts.length || posts.length')
      .goto(class:tinygoto='{tinygoto}', class:hide="{uiVisible == false}", bind:clientWidth='{gotoElWidth}')
        .btnwrapper
          span.btn.playpause.tooltip(
            data-tooltip="{autoplaystr}",
            class:play="{$autoplay}",
            on:click="{function(){toggleAutoPlay()}}"
          )
            Icon(icon="{$autoplay ? faPause : faPlay}")
          span.btn.download.tooltip(
            on:click="{function(){downloadFiles()}}",
            data-tooltip="{downloadstr}",
            class:dlready="{numSelected}"
          )
            Icon(icon="{faDownload}")
          span.btn.filter.tooltip(
            class:filterExpanded="{filterExpanded}",
            on:click="{function(){toggleFilter()}}",
            data-tooltip="Filter ( / )",
            bind:this='{filterRef}'
            class:dlready="{numSelected}"
          )
            +if('filterExpanded')
              input(bind:value='{filterValue}', on:click|stopPropagation, on:keydown|stopPropagation, type="text")
              +else
                Icon(icon="{faSearch}")
          span.btn.over18wrapper.tooltip(
            data-tooltip="{over18str}",
            class:over18="{!$over18}",
            on:click="{function(){toggleOver18()}}"
          )
            p nsfw
        .numswrapper
          +each('displayposts as post, i')
            span.nums(
              class:currnum="{index === i}",
              class:selected="{displayposts[i].selected}",
              class:over18="{displayposts[i].over18}",
              on:click="{function(){goto(i)}}"
            )
              img.small(alt="foo", src="{displayposts[i].preview.img.default}")
              p.small(class:curr="{index === i}") {i+1}
          +if('filterValue')
            span.btn.deepsearch(on:click='{function() {gotoDeepSearch()}}')
              p deep search 🡒
              a(href="https://google.com") goog
              a(href="/r/foox") /r/foox

  .prefetch
    +each('nexturls as nexturl')
      img(alt="prefetch", src="{nexturl.preview.img.default}")
      +if('nexturl.is_video')
        video
          +if('nexturl.preview.vid.webm')
            source(src="{nexturl.preview.vid.webm}")
          +if('nexturl.preview.vid.mp4')
            source(src="{nexturl.preview.vid.mp4}")
</template>
