<script lang="ts">
    import { onMount } from "svelte";
    import chroma from "chroma-js"

    let gridX = 6;
    let gridY = 6;
    let maxGridWidth = 400;
    let gridBlockSize = Math.ceil(maxGridWidth / gridX);
    let source = $state({ x: 0, y: 0 });
    let sink = $state({ x: gridX - 1, y: gridY - 1 });

    let maxWaypoints = 2;
    let waypoints: { x: number; y: number }[] = $state([]);
    let allPositions: { x: number; y: number }[] = [];

    let path: { x: number; y: number }[] = $state([{ x: -10, y: -10 }]);
    let difficulties = [
      {gridX: 6, gridY: 6, waypoints: 2,},
      {gridX: 6, gridY: 6, waypoints: 3}
    ]
    
    let blocks: {origin: {x: number, y: number}, parts: {x: number, y: number}[], color: string}[] = $state([])

    function generateCorrectPath() {
      source.x = Math.floor(Math.random() * (gridX * .3));
      source.y = Math.floor(Math.random() * (gridY * .3));
        sink.x = Math.floor(Math.random() * (gridX * .3)) + Math.floor(gridX * .7);
        sink.y = Math.floor(Math.random() * (gridY * .3)) + Math.floor(gridY * .7);
        waypoints = [];
        allPositions = [];
        for (let x = 0; x < gridX; x++) {
            for (let y = 0; y < gridY; y++) {
                allPositions.push({ x, y });
            }
        }
        let possibleWaypoints = [...allPositions];
        possibleWaypoints.splice(
            possibleWaypoints.findIndex(
                (p) => p.x === source.x && p.y === source.y,
            ),
            1,
        );
        possibleWaypoints.splice(
            possibleWaypoints.findIndex(
                (p) => p.x === sink.x && p.y === sink.y,
            ),
            1,
        );
        //remove positions that lie on edges bc those can get messy
        for(let i = 0; i < gridX; i++) {
          let idxTop = possibleWaypoints.findIndex(p=>p.x == i && p.y == 0)
          if(idxTop != -1) {
            possibleWaypoints.splice(idxTop, 1)
          }
          let idxBot = possibleWaypoints.findIndex(p=>p.x == i && p.y == gridY - 1)
          if(idxBot != -1) {
            possibleWaypoints.splice(idxBot, 1)
          }
        }
        for(let i = 0; i < gridY; i++) {
          let idxTop = possibleWaypoints.findIndex(p=>p.x == 0 && p.y == i)
          if(idxTop != -1) {
            possibleWaypoints.splice(idxTop, 1)
          }
          let idxBot = possibleWaypoints.findIndex(p=>p.x == gridX - 1 && p.y == i)
          if(idxBot != -1) {
            possibleWaypoints.splice(idxBot, 1)
          }
        }

        for (let i = 0; i < maxWaypoints; i++) {
            let idx = Math.floor(Math.random() * possibleWaypoints.length);
            waypoints.push(possibleWaypoints[idx]);
            possibleWaypoints.splice(idx, 1);
        }

        let currentPosition = { ...source };
        path = [source];
        let visited = [source];
        let waypointsToHit = [...waypoints];
        // sort waypoints in order of distance from source;
        waypointsToHit.sort(
            (a, b) =>
                Math.sqrt(Math.pow(a.x, 2) + Math.pow(a.y, 2)) -
                Math.sqrt(Math.pow(b.x, 2) + Math.pow(b.y, 2)),
        );
        waypointsToHit.push({ ...sink });
        let nextWaypoint: { x: number; y: number };
        let distanceToNextWaypoint = Infinity;

        function getValidDirections(point: { x: number; y: number }) {
            let allDirections = [
                { x: 0, y: -1 },
                { x: 0, y: 1 },
                { x: 1, y: 0 },
                { x: -1, y: 0 },
            ];
            let validDirections = [];
            for (let i = 0; i < allDirections.length; i++) {
                let d = allDirections[i];
                let tP = { x: point.x + d.x, y: point.y + d.y };
                if (tP.x < 0 || tP.y < 0) {
                    continue;
                }
                if (tP.x >= gridX || tP.y >= gridY) {
                    continue;
                }
                let found = visited.findIndex(
                    (p) => p.x == tP.x && p.y == tP.y,
                );

                if (found != -1) {
                    continue;
                }
                validDirections.push(d);
            }
            return validDirections;
        }

        for (let i = 0; i < waypointsToHit.length; i++) {
            nextWaypoint = waypointsToHit[i];
            //we need to determine which direction off of currentPosition we can use to get to there.
            // we can probably simply just use the closest cardinal direction but i feel like itd be cooler
            // to evaluate which directions are POSSIBLE and then use randomness to choose a direction.
            
            for (let i = 0; i < gridX * gridY; i++) {
                let validDirections = getValidDirections(currentPosition);
                //test against each valid direction to see which is closest to the target position
                let closestDirection = { x: 0, y: 0 };
                let closestDistance = Infinity;
                validDirections.forEach((d) => {
                    let dist = Math.sqrt(
                        Math.pow(currentPosition.x + d.x - nextWaypoint.x, 2) +
                            Math.pow(
                                currentPosition.y + d.y - nextWaypoint.y,
                                2,
                            ),
                    );
                    if (dist < closestDistance) {
                        closestDistance = dist;
                        closestDirection = d;
                    }
                });

                let newPos = {
                    x: currentPosition.x + closestDirection.x,
                    y: currentPosition.y + closestDirection.y,
                };
                currentPosition.x += closestDirection.x;
                currentPosition.y += closestDirection.y;
                path.push({ ...currentPosition });
                visited.push({ ...currentPosition });
                if (newPos.x == nextWaypoint.x && newPos.y == nextWaypoint.y) {
                    break;
                }
                
                if(newPos.x == sink.x && newPos.y == sink.y) {
                  return true;
                }
            }
        }
        
        return path.findIndex(p=>p.x == sink.x && p.y == sink.y) != -1
        
    }
    
    function generateBlocks() {
      blocks  =[]
      let occupied: {x: number, y: number}[] = [
        {x: 0, y: 0},
        {x: 0, y: gridY - 1},
        {x: gridX - 1, y: gridY - 1},
        {x: gridX - 1, y: 0},
      ]
      //add the four corners to the blocks
      blocks.push(
        {origin: {x:0,y:0},
          parts: [],
          color: chroma.random()
        }
      )
      blocks.push(
        {origin: {x:0,y:gridY - 1},
          parts: [],
          color: chroma.random()
        }
      )
      blocks.push(
        {origin: {x:gridX - 1,y:gridY - 1},
          parts: [],
          color: chroma.random()
        }
      )
      blocks.push(
        {origin: {x:gridX - 1,y:0},
          parts: [],
          color: chroma.random()
        }
      )
      let potentialGrowthPoints = [...allPositions];
      //remove the points that are already in blocks and their surrounding positions
      blocks.forEach(b=>{
        let p = b.origin;
        for(let x = -1; x <= 1; x++) {
          for(let y = -1; y <= 1; y++) {
            let idx = potentialGrowthPoints.findIndex(
              e=> (p.x + x) == e.x && (p.y + y) == e.y
            )
            if(idx != -1) {
              potentialGrowthPoints.splice(idx, 1)
            }
          }
        }
      })
      console.log(potentialGrowthPoints)
      let amountOfBlocks = Math.ceil(gridX * gridY / 4)
      for(let i = 0; i < amountOfBlocks - 4; i++) {
        let p = potentialGrowthPoints[Math.floor(Math.random() * potentialGrowthPoints.length)];
        for(let x = -1; x <= 1; x++) {
          for(let y = -1; y <= 1; y++) {
            let idx = potentialGrowthPoints.findIndex(
              e=> (p.x + x) == e.x && (p.y + y) == e.y
            )
            if(idx != -1) {
              potentialGrowthPoints.splice(idx, 1)
            }
          }
        }
        blocks.push({
          origin: {...p},
          parts: [],
          color: chroma.random()
        })
        occupied.push({...p});
      }
      function getValidDirections(point: { x: number; y: number }) {
          let allDirections = [
              { x: 0, y: -1 },
              { x: 0, y: 1 },
              { x: 1, y: 0 },
              { x: -1, y: 0 },
          ];
          let validDirections = [];
          for (let i = 0; i < allDirections.length; i++) {
              let d = allDirections[i];
              let tP = { x: point.x + d.x, y: point.y + d.y };
              if (tP.x < 0 || tP.y < 0) {
                  continue;
              }
              if (tP.x >= gridX || tP.y >= gridY) {
                  continue;
              }
              let found = occupied.findIndex(
                  (p) => p.x == tP.x && p.y == tP.y,
              );

              if (found != -1) {
                  continue;
              }
              validDirections.push(d);
          }
          return validDirections;
      }
      
      //now i think we should randomly iterate through each block, but make sure that each iteration hits every block
      for(let i = 0; i < 10; i++) {
        let idxs = new Array(amountOfBlocks);
        for(let x = 0; x < amountOfBlocks; x++) {
          idxs[x]=x
        }
        for(let n = 0; n < amountOfBlocks; n++) {
          let idx = Math.floor(Math.random() * idxs.length);
          let block = blocks[idxs[idx]]
          idxs.splice(idx, 1);
          //now we should go thru each of block's members and get all of the potential valid growth cells in a list
          let validDirectionsFromOrigin = getValidDirections(block.origin);
          let validPositions: {x: number, y: number}[] = [];
          validDirectionsFromOrigin.forEach(v=>{
            validPositions.push({x: block.origin.x + v.x, y: block.origin.y + v.y})
          })
          block.parts.forEach(p=>{
            let vdir = getValidDirections(p);
            vdir.forEach(v=>{
              validPositions.push({x: p.x + v.x, y: p.y + v.y})
            })
          })
          console.log(validPositions)
          //now pi9ck a random valid Position and add it to parts
          let pos = validPositions[Math.floor(Math.random() * validPositions.length)];
          block.parts.push({...pos})
          occupied.push({...pos})
        }
        
      }
    }

    onMount(() => {
        generateCorrectPath();
        generateBlocks();
    });
    
    function stresstest() {
      let fails = 0
      let i = 0;
      let intrv = setInterval(()=>{
        let s = generateCorrectPath();
        if(!s) {
          fails++;
        }
        i++;
        if(i == 100) {
          clearInterval(intrv)
          console.log(fails)
        }
      }, 0)
      
    }
</script>

<div
    class="game-grid"
    style:width={`${gridBlockSize * gridX}px`}
    style:height={`${gridBlockSize * gridY}px`}
>
    <svg
        class="path-overlay"
        viewBox={`0 0 ${gridBlockSize * gridX} ${gridBlockSize * gridY}`}
    >
        <path
            d={`M ${path[0].x * gridBlockSize + gridBlockSize / 2} ${path[0].y * gridBlockSize + gridBlockSize / 2} ` +
                path
                    .map(
                        (p) =>
                            `L ${p.x * gridBlockSize + gridBlockSize / 2} ${p.y * gridBlockSize + gridBlockSize / 2} `,
                    )
                    .join(" ")}
            stroke-width="10px"
            stroke="orange"
            stroke-linecap="round"
            fill="transparent"
        ></path>
    </svg>
    {#each Array(gridY) as _, y}
        {#each Array(gridX) as _, x}
            {#key blocks}
            <div
                class="game-grid-block"
                class:source={source.x === x && source.y === y}
                class:sink={sink.x === x && sink.y === y}
                style:background={
                blocks.find((b)=>{
                  return (b.origin.x == x && b.origin.y == y) || (b.parts.findIndex((e)=>{
                    return (e.x == x && e.y == y)
                  } )!= -1)
                })?.color ?? "null"
                }
                style:grid-column={x + 1}
                style:grid-row={y + 1}
                style:width="{gridBlockSize}px"
                style:height="{gridBlockSize}px"
            ></div>{/key}
        {/each}
    {/each}
</div>
<button onclick={stresstest}>stress test</button>
<button onclick={generateCorrectPath}>generateCorrectPath</button>
<button onclick={generateBlocks}>generateBlocks</button>

<style>
    .game-grid {
        display: grid;
        position: relative;
    }
    .path-overlay {
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
        z-index: 99;
        z-index: 99999;
    }
    .game-grid-block {
        background: #e3e3e3;
        border-radius: 8px;
        border: 2px solid var(--body-background);
        box-sizing: border-box;
    }
    .game-grid-block.source {
        outline:4px solid  blue;
        z-index: 9999;
    }
    .game-grid-block.sink {
        outline:4px solid red;
        z-index: 9999;
    }
    .game-grid-block.waypoint {
        background: green;
    }
</style>
