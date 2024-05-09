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
        src: "https://cdn.rive.app/animations/vehicles.riv",
        // OR the path to a discoverable and public Rive asset
        // src: '/public/example.riv',
        canvas: document.getElementById("canvas"),
        autoplay: true,
        stateMachines: "bumpy",
        onLoad: () => {
          r.resizeDrawingSurfaceToCanvas();
        },
    });
</script>

this on?
