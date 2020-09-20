<script lang="ts">
  import { onMount } from 'svelte';

  enum OBJECT_TYPE {
    BOX = 'BOX',
  }

  let topDiv: HTMLElement;
  let canvas: HTMLCanvasElement;
  let ctx: CanvasRenderingContext2D;
  let mouseCoordinates = { x1: -1, y1: -1, x2: -1, y2: -1, drawing: false };
  let dragCoordinateOrigin = { x: 0, y: 0, initialDrag: true };
  let mouseDown = false;
  let fillColor = '#FFFFFF';
  let objectCount = 0;

  let selectedObjectId = null;
  let selectedObject = null;
  let objects = [];
  let undoneObjects = [];

  onMount(() => {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight - topDiv.offsetHeight;
    ctx = canvas.getContext('2d');
  });

  function handleBoxDrag(event) {
    if (!mouseDown) {
      return;
    }
    if (selectedObjectId !== null) {
      // select object on first render of drag
      if (!selectedObject) {
        const o = objects.find((o) => o.id === selectedObjectId);
        const newObjects = objects.filter((o) => o.id !== selectedObjectId);
        selectedObject = { ...o };
        objects = [...newObjects];
      }

      const initialDrag = dragCoordinateOrigin.initialDrag;
      // calculate click offset with x1,y1
      const xOffset = dragCoordinateOrigin.x - event.clientX;
      const yOffset =
        dragCoordinateOrigin.y - (event.clientY - topDiv.offsetHeight);
      if (!initialDrag) {
        mouseCoordinates = {
          x1: selectedObject.x - xOffset,
          y1: selectedObject.y - yOffset,
          x2: selectedObject.x + selectedObject.width - xOffset,
          y2: selectedObject.y + selectedObject.height - yOffset,
          drawing: true,
        };
        drawBox(true, true);
      }
      dragCoordinateOrigin.x = event.clientX;
      dragCoordinateOrigin.y = event.clientY - topDiv.offsetHeight;
      dragCoordinateOrigin.initialDrag = false;
    } else if (mouseCoordinates.drawing === false) {
      mouseCoordinates.x1 = event.clientX;
      mouseCoordinates.y1 = event.clientY - topDiv.offsetHeight;
      mouseCoordinates.drawing = true;
    } else if (mouseCoordinates.drawing === true) {
      mouseCoordinates.x2 = event.clientX;
      mouseCoordinates.y2 = event.clientY - topDiv.offsetHeight;
      drawBox(true, false);
    }
  }

  function drawBox(isPreview: boolean, isSelected: boolean) {
    let width = Math.abs(mouseCoordinates.x2 - mouseCoordinates.x1);
    let height = Math.abs(mouseCoordinates.y2 - mouseCoordinates.y1);
    let x =
      mouseCoordinates.x1 > mouseCoordinates.x2
        ? mouseCoordinates.x2
        : mouseCoordinates.x1;
    let y =
      mouseCoordinates.y1 > mouseCoordinates.y2
        ? mouseCoordinates.y2
        : mouseCoordinates.y1;

    const newObject = {
      type: OBJECT_TYPE.BOX,
      x,
      y,
      width,
      height,
      isPreview,
      fillColor,
      highlighted: isSelected,
      id: objectCount += 1,
    };
    objects = [...objects, newObject];
    if (isSelected) {
      selectedObject = newObject;
    }

    render();
  }

  function render() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    for (const o of objects) {
      ctx.beginPath();
      ctx.rect(o.x, o.y, o.width, o.height);
      ctx.fillStyle = o.fillColor;
      ctx.strokeStyle = o.highlighted ? 'red' : '#999999';
      ctx.fill();
      ctx.stroke();
    }
    objects = objects.filter((o) => !o.isPreview);
  }

  function handleKeydown(event) {
    const ctrlPressed = event.ctrlKey || event.metaKey;

    if (event.key === 'z' && event.shiftKey && ctrlPressed) {
      event.preventDefault();
      const o = undoneObjects.pop();
      if (o) {
        objects.push(o);
        objects = [...objects, o];
        render();
      }
    }
    // undo
    else if (event.key === 'z' && ctrlPressed) {
      event.preventDefault();
      const o = objects.pop();
      if (o) {
        undoneObjects = [...undoneObjects, o];
        render();
      }
    }
  }

  function handleMouseUp() {
    console.log(mouseCoordinates.drawing && mouseCoordinates.x1 > -1);
    if (mouseCoordinates.drawing && mouseCoordinates.x1 > -1) {
      drawBox(false, false);
    }
    mouseDown = false;

    // reset
    mouseCoordinates = { x1: -1, y1: -1, x2: -1, y2: -1, drawing: false };
    selectedObjectId = null;
    selectedObject = null;
    dragCoordinateOrigin = { x: 0, y: 0, initialDrag: true };
  }

  function handleMouseDown(event) {
    selectedObjectId = null;
    selectedObject = null;
    const x = event.clientX;
    const y = event.clientY - topDiv.offsetHeight;

    let inContainer = false;
    for (const o of objects) {
      const x2 = o.x + o.width;
      const y2 = o.y + o.height;

      const inHorizontalBounds = (x <= o.x && x >= x2) || (x >= o.x && x <= x2);
      const inVerticalBounds = (y <= o.y && y >= y2) || (y >= o.y && y <= y2);

      if (inHorizontalBounds && inVerticalBounds) {
        objects.forEach((obj) => (obj.highlighted = false));
        o.highlighted = true;
        inContainer = true;
        selectedObjectId = o.id;
      }
    }

    render();

    mouseDown = true;
  }

  function handleResize() {
    canvas.width = window.innerWidth;
    canvas.height = window.innerHeight - topDiv.offsetHeight;
    render();
  }

  function handleColorChange() {
    for (const o of objects) {
      if (o.highlighted) {
        o.fillColor = fillColor;
      }
    }
    render();
  }
</script>

<style>
  canvas {
    border-top: 1px solid red;
  }
  main {
    text-align: center;
    max-width: 240px;
  }

  h1 {
    color: #ff3e00;
    text-transform: uppercase;
    font-size: 4em;
    font-weight: 100;
    margin-top: 0;
  }

  p {
    margin-bottom: 0;
  }

  .main-div {
    padding-bottom: 1em;
    box-sizing: border-box;
  }

  @media (min-width: 640px) {
    main {
      max-width: none;
    }
  }
  .toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 1em;
  }
  .tool-option {
    border: 1px solid #999;
    padding: 0.625em;
    cursor: pointer;
  }
</style>

<svelte:window on:keydown={handleKeydown} on:resize={handleResize} />
<main>
  <div class="main-div" bind:this={topDiv}>
    <h1>Hello Drag & Drop</h1>
    <p>
      Visit the <a href="https://svelte.dev/tutorial">Svelte tutorial</a> to learn
      how to build Svelte apps.
    </p>
    <div class="toolbar">
      <div class="tool-option" style={'background:' + fillColor}>
        Color: <input on:change={handleColorChange} bind:value={fillColor} />
      </div>
      <div
        class="tool-option"
        on:click={() => {
          ctx.clearRect(0, 0, canvas.width, canvas.height);
          objects = [];
          undoneObjects = [];
        }}>
        Clear
      </div>
    </div>
  </div>
  <canvas
    bind:this={canvas}
    on:mouseup={handleMouseUp}
    on:mousedown={handleMouseDown}
    on:mousemove={handleBoxDrag} />
</main>
