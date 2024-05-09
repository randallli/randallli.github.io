---
layout: page
title: Experimental
subtitle: Trying out some things
redirect_from:
  - /experiment
---
Expect this to be broken

## Rive
* Can I make an animation appear here runing on the rive runtime?

<script src="https://unpkg.com/@rive-app/canvas@2.10.3"></script>

<canvas id="canvas" width="500" height="500"></canvas>

<script>
    const r = new rive.Rive({
        src: '../assets/rive/star_rating.riv',
        canvas: document.getElementById("canvas"),
        autoplay: true,
        stateMachines: "State Machine 1",
        onLoad: () => {
          r.resizeDrawingSurfaceToCanvas();
        },
    });
</script>

this on?
