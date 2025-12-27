<script lang="ts">
  import {portal} from './Portal.svelte'
  import {getTinyContextForce, useRow} from "@/lib/tinybase/tinybase-stores";
  import type {Id} from "tinybase";
  import type {Loop} from "@/lib/model";

  let {video = document.querySelector("video"), id}: {
    video?: HTMLVideoElement | null,
    id: Id
  } = $props()

  const store = getTinyContextForce('store')

  let loopStore = $derived(useRow<Loop>(store, 'loops', id))
  let loop = $derived($loopStore)
  let startTime = $derived(loop?.startTime)

  let duration = $derived(video?.duration as number)

  let left = $derived(loop && duration ? loop.startTime / duration * 100 : 0)
  let width = $derived(loop && duration ? (loop.endTime - loop.startTime) / duration * 100 : 0)

  function ticker() {
    if (!video || !loop) {
      if (!video) console.log("[ActiveLoop] No video element");
      if (!loop) console.log("[ActiveLoop] No loop data for id:", id);
      return
    }

    if (video.currentTime >= loop.endTime || video.ended) {
      const stateBefore = {
        currentTime: video.currentTime,
        paused: video.paused,
        ended: video.ended,
        readyState: video.readyState
      };

      console.log(`[ActiveLoop] Looping back attempt:`, stateBefore, `Target: ${loop.startTime}`);
      
      video.currentTime = loop.startTime
      
      // 動画が終了している、または停止している場合は再生を試みる
      if (video.ended || video.paused) {
        video.play().then(() => {
          console.log("[ActiveLoop] Play success, current:", video.currentTime);
        }).catch(e => {
          console.warn("[ActiveLoop] Play failed:", e);
        });
      }
    }
  }

  $effect(() => {
    if (!video || startTime === undefined) return
    console.log(`[ActiveLoop] Initial seek to startTime: ${startTime}`);
    video.currentTime = startTime
  })

  $effect(() => {
    if (!video) {
      console.log("[ActiveLoop] Effect: No video element found");
      return;
    }
    console.log("[ActiveLoop] Effect: Starting ticker and listeners");

    let frame: number;
    const check = () => {
      ticker();
      frame = requestAnimationFrame(check);
    }
    frame = requestAnimationFrame(check);

    const onEnded = () => {
      console.log("[ActiveLoop] Event: ended");
      ticker();
    };
    video.addEventListener("ended", onEnded)
    video.addEventListener("timeupdate", ticker)

    return () => {
      console.log("[ActiveLoop] Effect: Cleaning up");
      cancelAnimationFrame(frame);
      video.removeEventListener("ended", onEnded)
      video.removeEventListener("timeupdate", ticker)
    }
  })

  export function seekStart(preRoll = 0) {
    if (!video) return

    video.currentTime = Math.max(loop.startTime - preRoll, 0)
  }
</script>

<div class="bar" use:portal={{target: ".ytp-progress-bar"}} style:left={left + '%'} style:width={width + '%'}></div>

<style>

    .bar {
        position: absolute;
        height: 3px;
        background: rgb(5 255 0 / 50%);
        top: 1px;
        z-index: 100;
    }

</style>
