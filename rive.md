---
layout: page
title: Rive
subtitle: animations
redirect_from:
---
Here I have tried out some [rive](https://rive.app/) animations. Some are interactive.

## Triple Li logo

<script src="https://unpkg.com/@rive-app/canvas@2.10.3"></script>

<canvas id="3Li_Logo" width="500" height="500"></canvas>

<script>
    const r = new rive.Rive({
        src: '../assets/rive/tripleli_logo.riv',
        canvas: document.getElementById("3Li_Logo"),
        autoplay: true,
        stateMachines: "State Machine 1",
        onLoad: () => {
          r.resizeDrawingSurfaceToCanvas();
        },
    });
</script>

## Rive tutorial

<canvas id="canvas" width="100" height="100"></canvas>

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
