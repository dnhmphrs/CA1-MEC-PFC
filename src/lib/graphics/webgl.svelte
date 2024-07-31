<script>
  import { onMount } from 'svelte';

  import { setupBackground, drawBackground, cleanupBackground } from './webgl/Background.js';

  let canvas;
  let aspectRatio;
  let mouseX = 0;
  let mouseY = 0;
  let globalConfig = { /* your global config */ };

  const borderSize = 2;

  const layoutConfig = [
    { width: 0.33, grid: [5, 5] },
    { width: 0.33, grid: [5, 5] },
    { width: 0.34, grid: [1, 15] }
  ];

  function calculatePerfectHeight(canvasHeight, gridRows, borderSize, initialHeightPercentage) {
  const totalBorders = (gridRows - 1) * borderSize;
  const availableHeight = Math.floor((canvasHeight * initialHeightPercentage) - totalBorders);
  const totalHeightPerCell = Math.floor(availableHeight / gridRows);
  const totalHeight = (totalHeightPerCell * gridRows) + totalBorders;

  return totalHeight / canvasHeight;
}


  layoutConfig.forEach((config) => {
    config.height = calculatePerfectHeight(window.innerHeight, config.grid[0], borderSize, 0.95);
  });

  function updateMousePosition(event) {
    const rect = canvas.getBoundingClientRect();
    mouseX = ((event.clientX - rect.left) / rect.width) * 2 - 1;
    mouseY = ((event.clientY - rect.top) / rect.height) * -2 + 1;
  }

  onMount(() => {
    const gl = canvas.getContext('webgl');
    if (!gl) {
      console.error('Unable to initialize WebGL.');
      return;
    }

    function resizeCanvasToDisplaySize(canvas, multiplier = 1) {
      const width = canvas.clientWidth * multiplier | 0;
      const height = canvas.clientHeight * multiplier | 0;

      if (canvas.width !== width || canvas.height !== height) {
        canvas.width = width;
        canvas.height = height;
        return true;
      }

      return false;
    }

    function resizeCanvas() {
      if (resizeCanvasToDisplaySize(canvas, window.devicePixelRatio)) {
        aspectRatio = canvas.width / canvas.height;
        gl.viewport(0, 0, canvas.width, canvas.height);
      }
    }

    window.addEventListener('resize', resizeCanvas);
    resizeCanvas(); // Initial resize

    const shader1 = setupBackground(gl);
    const shader2 = setupBackground(gl);
    const shader3 = setupBackground(gl);

    canvas.addEventListener('mousemove', updateMousePosition);

    function render() {
      gl.clear(gl.COLOR_BUFFER_BIT);
      const now = performance.now();

      let offsetX = 0;

      layoutConfig.forEach((config, index) => {
        const classWidth = Math.floor(canvas.width * config.width);
        const classHeight = Math.floor(canvas.height * config.height);
        const [gridCols, gridRows] = config.grid;
        const totalBorderWidth = (gridCols - 1) * borderSize;
        const totalBorderHeight = (gridRows - 1) * borderSize;
        const cellWidth = Math.floor((classWidth - totalBorderWidth) / gridCols);
        const cellHeight = Math.floor((classHeight - totalBorderHeight) / gridRows);

        for (let row = 0; row < gridRows; row++) {
          for (let col = 0; col < gridCols; col++) {
            const x = offsetX + col * (cellWidth + borderSize);
            const y = row * (cellHeight + borderSize);
            gl.viewport(x, y, cellWidth, cellHeight);

            if (index === 0) {
              drawBackground(gl, shader1, now, aspectRatio);
            } else if (index === 1) {
              drawBackground(gl, shader2, now, aspectRatio);
            } else if (index === 2) {
              drawBackground(gl, shader3, now, aspectRatio);
            }
          }
        }

        offsetX += classWidth + borderSize;
      });

      requestAnimationFrame(render);
    }

    requestAnimationFrame(render);
    canvas.style.opacity = 1;

    return () => {
      window.removeEventListener('resize', resizeCanvas);
      canvas.removeEventListener('mousemove', updateMousePosition);
      cleanupBackground(gl, shader1);
      cleanupBackground(gl, shader2);
      cleanupBackground(gl, shader3);
    };
  });
</script>

<canvas bind:this={canvas} class:geometry={true}></canvas>

<style>
.geometry {
  position: absolute;
  top: 0;
  left:0;
  width: 100%;
  height: 100%;
  display: block;
  padding: 0;
  margin: 0;
  border: none;
  z-index: -1;

  /* animations */
  opacity: 0; /* start invisible */
  transition: opacity 0.5s ease-in-out;
}
</style>
