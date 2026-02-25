<script lang="ts">
    import { onMount, tick } from "svelte";
    import chroma, { type Color } from "chroma-js";
    import DTLogo from "$lib/dailytrojan-lib/Games_Logo.svg";
    import { DTGameCore } from "$lib/dailytrojan-lib/gameCore";
    import { Spring, Tween } from "svelte/motion";
    import { sineInOut } from "svelte/easing";
    let gameSplash: HTMLElement | null = null;
    let gameDate: HTMLElement | null = null;

    let DTGCore: DTGameCore;
    let isWebKit = $state(false);
    
    let {difficulty}: {
      difficulty: "easy" | "medium" | "hard"
    } = $props()

    let gridX = $state(6);
    let gridY = $state(6);
    let maxGridWidth = $state(400);
    let gridBlockWidth = $derived(Math.ceil(maxGridWidth / gridX));
    let gridBlockHeight = $derived(gridBlockWidth);
    let source = $state({ x: 0, y: 0 });
    let sink = $state({ x: gridX - 1, y: gridY - 1 });

    let amountOfBlocks = $state(0);
    let splashReady = $state(false);

    let maxWaypoints = 2;
    let waypoints: { x: number; y: number }[] = $state([]);
    let allPositions: { x: number; y: number }[] = [];

    let path: { x: number; y: number }[] = $state([{ x: -10, y: -10 }]);
    let difficulties = {
      "easy": { gridX: 6, gridY: 6, waypoints: 2, rdx: 1 },
        "medium": { gridX: 7, gridY: 7, waypoints: 6, rdx: 8 },
        "hard": { gridX: 9, gridY: 8, waypoints: 8, rdx: 24 },
        "insane": { gridX: 8, gridY: 8, waypoints: 12, rdx: 55 },
    };
    type Block = {
        position: { x: number; y: number };
        truthPosition: { x: number; y: number };
        placed: boolean;
        locked: boolean;
        parts: { x: number; y: number }[];
        color: Color;
        id: string;
        dragging: boolean;
        animDelay: string;
        blockHome: HTMLElement | null;
        blockEl: HTMLElement | null;
        isCurrentlyAnimating: boolean;
        cssPosition: Spring<{ x: number; y: number }>;
        cardinalTwoPairs: { x1: number; y1: number; x2: number; y2: number }[]; // pairs of cardinal parts that are both in the block, used for validation
        signalPath: { x: number; y: number }[]; //the part of the signal path that is contained in this block, used for validation
    };
    let blocks: Block[] = $state([]);
    let gameBlockSelectors: HTMLElement | null = $state(null);

    onMount(async () => {
        initializeGame(difficulties[difficulty]);
        window.initGame = initializeGame;
        //@ts-ignore
        window.gameCompleteEffect = gameCompleteEffect;
        //@ts-ignore
        isWebKit = //@ts-ignore
            typeof window.webkitConvertPointFromNodeToPage === "function" ||
            "webkitRequestAnimationFrame" in window;
    });
    async function initializeGame(difficulty: {
        gridX: number;
        gridY: number;
        waypoints: number;
        rdx: number;
    }) {
      disableInput = false;
        gridX = difficulty.gridX;
        gridY = difficulty.gridY;
        maxWaypoints = difficulty.waypoints;
        DTGCore = new DTGameCore(gameSplash, gameDate);
        for (let i = 0; i < difficulty.rdx; i++) {
            DTGCore.randomFloat();
        }
        window.DTGCore = DTGCore;

        for (let i = 0; i < gridX; i++) {
            occupiedSpaces[i] = new Array(gridY).fill(false);
        }
        for (let i = 0; i < gridX; i++) {
            hoveringSpaces[i] = new Array(gridY).fill(false);
        }

        generateCorrectPath();
        generateBlocks();
        await tick();
        setBlockDefaultPositions();
        await tick();
        onResize();
        splashReady = true;
    }

    function genRandomId() {
        return DTGCore.randomFloat().toString(36).substr(2, 9);
    }

    function generateCorrectPath() {
        source.x = Math.floor(DTGCore.randomFloat() * (gridX * 0.3));
        source.y = Math.floor(DTGCore.randomFloat() * (gridY * 0.3));
        sink.x =
            Math.floor(DTGCore.randomFloat() * (gridX * 0.3)) +
            Math.floor(gridX * 0.7);
        sink.y =
            Math.floor(DTGCore.randomFloat() * (gridY * 0.3)) +
            Math.floor(gridY * 0.7);
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
        for (let i = 0; i < gridX; i++) {
            let idxTop = possibleWaypoints.findIndex(
                (p) => p.x == i && p.y == 0,
            );
            if (idxTop != -1) {
                possibleWaypoints.splice(idxTop, 1);
            }
            let idxBot = possibleWaypoints.findIndex(
                (p) => p.x == i && p.y == gridY - 1,
            );
            if (idxBot != -1) {
                possibleWaypoints.splice(idxBot, 1);
            }
        }
        for (let i = 0; i < gridY; i++) {
            let idxTop = possibleWaypoints.findIndex(
                (p) => p.x == 0 && p.y == i,
            );
            if (idxTop != -1) {
                possibleWaypoints.splice(idxTop, 1);
            }
            let idxBot = possibleWaypoints.findIndex(
                (p) => p.x == gridX - 1 && p.y == i,
            );
            if (idxBot != -1) {
                possibleWaypoints.splice(idxBot, 1);
            }
        }

        for (let i = 0; i < maxWaypoints; i++) {
            let idx = Math.floor(
                DTGCore.randomFloat() * possibleWaypoints.length,
            );
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

                if (newPos.x == sink.x && newPos.y == sink.y) {
                    return true;
                }
            }
        }

        if (path.findIndex((p) => p.x == sink.x && p.y == sink.y) == -1) {
            //set sink to the last point just bc like if it fails like....
            let p = path[path.length - 1];
            sink.x = p.x;
            sink.y = p.y;
        }

        return path.findIndex((p) => p.x == sink.x && p.y == sink.y) != -1;
    }

    let springProperties: any = {
        precision: 0.1,
    };

    function generateBlocks() {
        blocks = [];
        let occupied: { x: number; y: number }[] = [
            { x: 0, y: 0 },
            { x: 0, y: gridY - 1 },
            { x: gridX - 1, y: gridY - 1 },
            { x: gridX - 1, y: 0 },
        ];
        amountOfBlocks = Math.ceil((gridX * gridY) / 4);
        let colors = generateBeautifulColors(amountOfBlocks, {
            hueStep: 15,
            hueJitter: 12,
            lightRange: [0.35, 0.8],
            satRange: [0.5, 0.8],
        });
        //add the four corners to the blocks
        blocks.push({
            position: { x: 0, y: 0 },
            truthPosition: { x: 0, y: 0 },
            parts: [],
            color: colors[0],
            signalPath: [],
            dragging: false,
            cardinalTwoPairs: [],
            id: genRandomId(),
            placed: false,
            animDelay: `${Math.random() * 4}s`,
            locked: false,
            blockHome: null,
            blockEl: null,
            isCurrentlyAnimating: false,
            cssPosition: new Spring({ x: 0, y: 0 }, springProperties),
        });
        blocks.push({
            position: { x: 0, y: gridY - 1 },
            truthPosition: { x: 0, y: gridY - 1 },
            parts: [],
            color: colors[1],
            signalPath: [],
            dragging: false,
            animDelay: `${Math.random() * 4}s`,
            cardinalTwoPairs: [],
            id: genRandomId(),
            placed: false,
            isCurrentlyAnimating: false,
            locked: false,
            blockHome: null,
            blockEl: null,
            cssPosition: new Spring({ x: 0, y: 0 }, springProperties),
        });
        blocks.push({
            position: { x: gridX - 1, y: gridY - 1 },
            truthPosition: { x: gridX - 1, y: gridY - 1 },
            parts: [],
            color: colors[2],
            signalPath: [],
            cardinalTwoPairs: [],
            id: genRandomId(),
            dragging: false,
            placed: false,
            animDelay: `${Math.random() * 4}s`,
            locked: false,
            blockEl: null,
            isCurrentlyAnimating: false,
            blockHome: null,
            cssPosition: new Spring({ x: 0, y: 0 }, springProperties),
        });
        blocks.push({
            position: { x: gridX - 1, y: 0 },
            truthPosition: { x: gridX - 1, y: 0 },
            parts: [],
            dragging: false,
            color: colors[3],
            signalPath: [],
            cardinalTwoPairs: [],
            isCurrentlyAnimating: false,
            id: genRandomId(),
            animDelay: `${Math.random() * 4}s`,
            placed: false,
            blockEl: null,
            locked: false,
            blockHome: null,
            cssPosition: new Spring({ x: 0, y: 0 }, springProperties),
        });
        let potentialGrowthPoints = [...allPositions];
        //remove the points that are already in blocks and their surrounding positions
        blocks.forEach((b) => {
            let p = b.position;
            for (let x = -1; x <= 1; x++) {
                for (let y = -1; y <= 1; y++) {
                    let idx = potentialGrowthPoints.findIndex(
                        (e) => p.x + x == e.x && p.y + y == e.y,
                    );
                    if (idx != -1) {
                        potentialGrowthPoints.splice(idx, 1);
                    }
                }
            }
        });
        for (let i = 0; i < amountOfBlocks - 4; i++) {
            let p =
                potentialGrowthPoints[
                    Math.floor(
                        DTGCore.randomFloat() * potentialGrowthPoints.length,
                    )
                ];
            for (let x = -1; x <= 1; x++) {
                for (let y = -1; y <= 1; y++) {
                    let idx = potentialGrowthPoints.findIndex(
                        (e) => p.x + x == e.x && p.y + y == e.y,
                    );
                    if (idx != -1) {
                        potentialGrowthPoints.splice(idx, 1);
                    }
                }
            }
            if (p == undefined) continue;
            blocks.push({
                position: { ...p },
                truthPosition: { ...p },
                parts: [],
                color: colors[i + 4],
                signalPath: [],
                cardinalTwoPairs: [],
                id: genRandomId(),
                placed: false,
                dragging: false,
                animDelay: `${Math.random() * 4}s`,
                locked: false,
                blockHome: null,
                blockEl: null,
                isCurrentlyAnimating: false,
                cssPosition: new Spring({ x: 0, y: 0 }, springProperties),
            });
            occupied.push({ ...p });
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
        for (let i = 0; i < 10; i++) {
            let idxs = new Array(blocks.length);
            for (let x = 0; x < blocks.length; x++) {
                idxs[x] = x;
            }
            for (let n = 0; n < blocks.length; n++) {
                let idx = Math.floor(DTGCore.randomFloat() * idxs.length);
                let block = blocks[idxs[idx]];
                idxs.splice(idx, 1);
                //now we should go thru each of block's members and get all of the potential valid growth cells in a list
                let validDirectionsFromOrigin = getValidDirections(
                    block.position,
                );
                let validPositions: { x: number; y: number }[] = [];
                validDirectionsFromOrigin.forEach((v) => {
                    validPositions.push({
                        x: block.position.x + v.x,
                        y: block.position.y + v.y,
                    });
                });
                block.parts.forEach((p) => {
                    let vdir = getValidDirections({
                        x: block.position.x + p.x,
                        y: block.position.y + p.y,
                    });
                    vdir.forEach((v) => {
                        validPositions.push({
                            x: block.position.x + p.x + v.x,
                            y: block.position.y + p.y + v.y,
                        });
                    });
                });
                //now pi9ck a random valid Position and add it to parts
                if (validPositions.length == 0) {
                    continue;
                }
                let pos =
                    validPositions[
                        Math.floor(
                            DTGCore.randomFloat() * validPositions.length,
                        )
                    ];
                block.parts.push({
                    x: pos.x - block.position.x,
                    y: pos.y - block.position.y,
                });
                occupied.push({ ...pos });
            }
        }
        //now we need to add the origin back to the parts
        blocks.forEach((b) => {
            b.parts.push({ x: 0, y: 0 });
        });
        //now we need to go through each part of the blcok, including the origin, and for each of them see if there is any cardinal pair, that being a cell of this block that is either up down left or right

        blocks.forEach((b) => {
            let bounds = boundsOfArray(b.parts);
            b.parts.forEach((p) => {
                p.x -= bounds.relative.minX;
                p.y -= bounds.relative.minY;
            });
            b.position.x += bounds.relative.minX;
            b.position.y += bounds.relative.minY;
            b.truthPosition.x += bounds.relative.minX;
            b.truthPosition.y += bounds.relative.minY;
        });

        blocks.forEach((b) => {
            let cardinalTwoPairs: {
                x1: number;
                y1: number;
                x2: number;
                y2: number;
            }[] = [];
            let allParts = [...b.parts];
            allParts.forEach((p) => {
                let cardinalDirections = [
                    { x: 0, y: -1 },
                    { x: 0, y: 1 },
                    { x: 1, y: 0 },
                    { x: -1, y: 0 },
                ];
                cardinalDirections.forEach((d) => {
                    let match = allParts.find(
                        (part) => part.x == p.x + d.x && part.y == p.y + d.y,
                    );
                    if (match) {
                        //we have a cardinal pair! we should add it to the block object as a pair of absolute positions
                        cardinalTwoPairs.push({
                            x1: p.x,
                            y1: p.y,
                            x2: match.x,
                            y2: match.y,
                        });
                    }
                });
            });
            b.cardinalTwoPairs = cardinalTwoPairs;
        });

        //now we need to get the signal path segments that are contained inside of each block
        blocks.forEach((b) => {
            let blockSignalPath: { x: number; y: number }[] = [];
            path.forEach((p) => {
                let relativeX = p.x - b.position.x;
                let relativeY = p.y - b.position.y;
                if (relativeX == 0 && relativeY == 0) {
                    blockSignalPath.push({ x: relativeX, y: relativeY });
                }
                b.parts.forEach((part) => {
                    if (relativeX == part.x && relativeY == part.y) {
                        blockSignalPath.push({ x: relativeX, y: relativeY });
                    }
                });
            });
            b.signalPath = blockSignalPath;
        });

        //now we need to place and lock the blocks that contain the source and the sink;
        blocks.forEach((b) => {
            b.parts.forEach((p) => {
                let partPos = {
                    x: p.x + b.position.x,
                    y: p.y + b.position.y,
                };
                if (
                    (partPos.x == source.x && partPos.y == source.y) ||
                    (partPos.x == sink.x && partPos.y == sink.y)
                ) {
                    b.locked = true;
                    //bounds.relative.minY * gridBlockHeight
                    placeBlock(b, b.position, true);
                }
            });
        });
    }

    let selectedBlock: Block | null;
    let pPointer: { x: number; y: number } = { x: 0, y: 0 };
    let dragSway = $state(0);
    let activelyDragging = $state(false);
    function blockPointerDown(block: Block, e: PointerEvent) {
        e.preventDefault();
        e.stopPropagation();
        dragSway = 0;
        if (!block) return;
        if (block.locked) return;
        if (activelyDragging) return;
        selectedBlock = block;
        activelyDragging = true;
        (e.target as HTMLElement).setPointerCapture(e.pointerId);

        // Disable scrolling on the container during drag
        if (gameBlockSelectors) {
            gameBlockSelectors.style.overflowX = "hidden";
        }

        if (
            selectedBlock != null &&
            !selectedBlock.locked &&
            !selectedBlock.placed &&
            selectedBlock.blockHome != null &&
            gameGrid != null &&
            selectedBlock.blockEl != null
        ) {
            //set the blocks position to the center of its blockhome;
            let rect = selectedBlock.blockHome.getBoundingClientRect();
            let bRect = selectedBlock.blockEl.getBoundingClientRect();
            let grid = gameGrid.getBoundingClientRect();
            selectedBlock.cssPosition.set(
                {
                    x: rect.left - grid.left + rect.width / 2 - bRect.width / 2,
                    y: rect.top - grid.top + rect.height / 2 - bRect.height / 2,
                },
                { instant: true },
            );
        }

        selectedBlock.isCurrentlyAnimating = true;

        pPointer = { x: e.clientX, y: e.clientY };

        for (let x = 0; x < gridX; x++) {
            for (let y = 0; y < gridY; y++) {
                occupiedSpaces[x][y] = false;
            }
        }
        selectedBlock.dragging = true;
        blocks.forEach((b) => {
            if (b.placed && b != selectedBlock) {
                b.parts.forEach((p) => {
                    occupiedSpaces[b.position.x + p.x][b.position.y + p.y] =
                        true;
                });
            }
        });
    }

    function pointerMove(e: PointerEvent) {
        if (!activelyDragging) return;
        e.preventDefault();
        if (
            !(e.target as HTMLElement).hasPointerCapture(e.pointerId) ||
            gameGrid == null ||
            selectedBlock == null ||
            selectedBlock.blockEl == null
        )
            return;
        currentlyHovering = true;
        let current = selectedBlock.cssPosition.current;
        let bRect = selectedBlock.blockEl.getBoundingClientRect();
        let grid = gameGrid.getBoundingClientRect();
        selectedBlock.cssPosition.set({
            x: e.clientX - grid.left - bRect.width / 2,
            y: e.clientY - grid.top - bRect.height / 2,
        });
        selectedBlock.isCurrentlyAnimating = true;
        let sign = Math.sign(e.clientX - pPointer.x);
        dragSway = Math.min(Math.abs(e.clientX - pPointer.x), 26) * sign;
        pPointer = { x: e.clientX, y: e.clientY };
        /// now calculating the hovering spaces
        for (let x = 0; x < gridX; x++) {
            for (let y = 0; y < gridY; y++) {
                hoveringSpaces[x][y] = false;
            }
        }

        let x = e.clientX - grid.left;
        let y = e.clientY - grid.top;
        let bounds = boundsOfArray(selectedBlock.parts);
        x -= (bounds.absolute.maxX * gridBlockWidth) / 2;
        y -= (bounds.absolute.maxY * gridBlockWidth) / 2;
        x /= gridBlockWidth;
        y /= gridBlockHeight;
        x = Math.floor(x);
        y = Math.floor(y);
        let valid = validateBlockPlacement(selectedBlock, { x, y });
        currentHoveringPlacement = { x, y };
        currentPlacementInvalid = !valid;
        selectedBlock.parts.forEach((p) => {
            if (
                x + p.x >= 0 &&
                x + p.y >= 0 &&
                x + p.x < gridX &&
                y + p.y < gridY
            )
                hoveringSpaces[x + p.x][y + p.y] = true;
        });
    }
    let currentPlacementInvalid = $state(false);
    let currentHoveringPlacement: { x: number; y: number } = $state({
        x: 0,
        y: 0,
    });
    let currentlyHovering = $state(false);
    function pointerUp(e: PointerEvent) {
        if (!activelyDragging) return;
        activelyDragging = false;
        dragSway = 0;
        currentlyHovering = false;
        e.preventDefault();
        (e.target as HTMLElement).releasePointerCapture(e.pointerId);

        // Re-enable scrolling on the container
        if (gameBlockSelectors) {
            gameBlockSelectors.style.overflowX = "auto";
        }
        if (
            selectedBlock != null &&
            !selectedBlock.locked &&
            !selectedBlock.placed &&
            selectedBlock.blockHome != null &&
            gameGrid != null &&
            selectedBlock.blockEl != null
        ) {
            //set the blocks position to the center of its blockhome;
            let rect = selectedBlock.blockHome.getBoundingClientRect();
            let bRect = selectedBlock.blockEl.getBoundingClientRect();
            let grid = gameGrid.getBoundingClientRect();
            selectedBlock.cssPosition.set({
                x: rect.left - grid.left + rect.width / 2 - bRect.width / 2,
                y: rect.top - grid.top + rect.height / 2 - bRect.height / 2,
            });
        }

        if (selectedBlock == null) return;

        selectedBlock.dragging = false;
        if (gameGrid == null) return;
        let grid = gameGrid.getBoundingClientRect();
        let x = e.clientX - grid.left;
        let y = e.clientY - grid.top;
        let bounds = boundsOfArray(selectedBlock.parts);
        x -= (bounds.absolute.maxX * gridBlockWidth) / 2;
        y -= (bounds.absolute.maxY * gridBlockWidth) / 2;
        x /= gridBlockWidth;
        y /= gridBlockHeight;
        x = Math.floor(x);
        y = Math.floor(y);
        let valid = validateBlockPlacement(selectedBlock, { x, y });
        if (valid) {
            placeBlock(selectedBlock, { x, y });
        } else {
            selectedBlock.placed = false;
            if (
                !selectedBlock.locked &&
                !selectedBlock.placed &&
                selectedBlock.blockHome != null &&
                gameGrid != null &&
                selectedBlock.blockEl != null
            ) {
                //set the blocks position to the center of its blockhome;
                let rect = selectedBlock.blockHome.getBoundingClientRect();
                let bRect = selectedBlock.blockEl.getBoundingClientRect();
                let grid = gameGrid.getBoundingClientRect();
                selectedBlock.placed = false;
                deAnimateSelectedBlock(selectedBlock, {
                    x: rect.left - grid.left + rect.width / 2 - bRect.width / 2,
                    y: rect.top - grid.top + rect.height / 2 - bRect.height / 2,
                });
            }
        }
        selectedBlock = null;
    }

    async function deAnimateSelectedBlock(
        block: Block,
        pos: { x: number; y: number },
    ) {
        await block.cssPosition.set(pos);
        block.isCurrentlyAnimating = false;
    }

    function playGame() {
        DTGCore.hideSplashScreen();

        // if (gameOver) {
        //     finishGame();
        // }
        // let firstTime = localStorage.getItem("firstTimePlayed") == null;
        // if (firstTime) {
        //     localStorage.setItem("firstTimePlayed", "true");
        //     showHowTo = true;
        // }
    }
    let occupiedSpaces: boolean[][] = $state(new Array(gridX));

    let hoveringSpaces: boolean[][] = $state(new Array(gridX));

    function validateBlockPlacement(
        block: Block,
        position: { x: number; y: number },
    ) {
        for (let i = 0; i < block.parts.length; i++) {
            let p = block.parts[i];

            if (position.x + p.x < 0 || position.y + p.y < 0) {
                return false;
            }
            if (position.x + p.x >= gridX || position.y + p.y >= gridY) {
                return false;
            }
            if (occupiedSpaces[position.x + p.x][position.y + p.y]) {
                return false;
            }
        }
        return true;
    }

    function placeBlock(
        block: Block,
        position: { x: number; y: number },
        instant = false,
    ) {
        block.placed = true;
        block.position = { ...position };
        block.cssPosition.set(
            {
                x: position.x * gridBlockWidth,
                y: position.y * gridBlockHeight,
            },
            { instant: instant },
        );
        //need to revalidate the occupied spaces!
        for (let x = 0; x < gridX; x++) {
            for (let y = 0; y < gridY; y++) {
                occupiedSpaces[x][y] = false;
            }
        }
        blocks.forEach((b) => {
            if (b.placed) {
                b.parts.forEach((p) => {
                    occupiedSpaces[b.position.x + p.x][b.position.y + p.y] =
                        true;
                });
            }
        });
        console.log(validateWin());
        if (validateWin()) {
            gameCompleteEffect();
        }
    }

    function repositionBlocks() {
        blocks.forEach((b) => {
            if (b.placed)
                b.cssPosition.set(
                    {
                        x: b.position.x * gridBlockWidth,
                        y: b.position.y * gridBlockHeight,
                    },
                    { instant: true },
                );
        });
    }

    let gameGrid: HTMLElement | null = $state(null);
    function setBlockDefaultPositions() {
        blocks.forEach((block) => {
            if (
                !block.locked &&
                !block.placed &&
                block.blockHome != null &&
                gameGrid != null &&
                block.blockEl != null
            ) {
                //set the blocks position to the center of its blockhome;
                let rect = block.blockHome.getBoundingClientRect();
                let bRect = block.blockEl.getBoundingClientRect();
                let grid = gameGrid.getBoundingClientRect();
                block.cssPosition.set(
                    {
                        x:
                            rect.left -
                            grid.left +
                            rect.width / 2 -
                            bRect.width / 2,
                        y:
                            rect.top -
                            grid.top +
                            rect.height / 2 -
                            bRect.height / 2,
                    },
                    { instant: true },
                );
            }
        });
    }

    function boundsOfArray(arr: { x: number; y: number }[]) {
        let minX = Infinity;
        let maxX = -Infinity;
        let minY = Infinity;
        let maxY = -Infinity;
        arr.forEach((p) => {
            if (p.x < minX) {
                minX = p.x;
            }
            if (p.x > maxX) {
                maxX = p.x;
            }
            if (p.y < minY) {
                minY = p.y;
            }
            if (p.y > maxY) {
                maxY = p.y;
            }
        });
        //also return bounds with min at 0, adjusting max accordingly
        return {
            relative: { minX, maxX, minY, maxY },
            absolute: {
                minX: 0,
                maxX: maxX - minX,
                minY: 0,
                maxY: maxY - minY,
            },
        };
    }

    function generateBeautifulColors(
        count: number,
        {
            satRange = [0.5, 0.85], // keep it vivid but not nuclear
            lightRange = [0.35, 0.75], // avoid muddy darks + washed whites
            hueJitter = 8, // degrees
            hueStep = 360 / count, // base step to ensure even hue distribution
            satJitter = 0.05,
            lightJitter = 0.05,
            sortByLightness = false,
        } = {},
    ) {
        const clamp = (v: number, a: number, b: number) =>
            Math.max(a, Math.min(b, v));
        const rand = (a: number, b: number) =>
            a + DTGCore.randomFloat() * (b - a);

        const baseHue = rand(0, 360);
        const step = 360 / count;

        const colors = [];

        for (let i = 0; i < count; i++) {
            const hue =
                (baseHue + i * hueStep + rand(-hueJitter, hueJitter) + 360) %
                360;
            const sat = clamp(
                rand(satRange[0], satRange[1]) + rand(-satJitter, satJitter),
                0,
                1,
            );
            const light = clamp(
                rand(lightRange[0], lightRange[1]) +
                    rand(-lightJitter, lightJitter),
                0,
                1,
            );

            colors.push(chroma.hsl(hue, sat, light));
        }

        if (sortByLightness) {
            colors.sort((a, b) => a.luminance() - b.luminance());
        }

        return colors;
    }

    //validate win.
    // we need to check that any blocks that have path elements in them are in the correct position
    // that is all
    // we then just need to check and make sure that al blocks are placed somewhere.
    function validateWin() {
        let placedBlocksValid = true;
        blocks.forEach((b) => {
            if (!b.placed) {
                placedBlocksValid = false;
            }
            if (b.signalPath.length > 0) {
                if (
                    b.truthPosition.x != b.position.x ||
                    b.truthPosition.y != b.position.y
                ) {
                    placedBlocksValid = false;
                }
            }
        });
        return placedBlocksValid;
    }

    let strokePadding = 4;
    let blockRoundness = $state(12);

    let previewHoverPadding = 0;
    let previewOutlineWidth = 4;
    let verticalBlockPadding = $state(7);
    let horizontalBlockPadding = $state(2);
    let blockPathWidth = $state(16);

    async function onResize() {
        if (window.innerWidth <= 500 && maxGridWidth != 300) {
            maxGridWidth = 300;
            verticalBlockPadding = 6;
            horizontalBlockPadding = 1;
            await tick();
            await tick();
            await tick();
            await tick();
            setBlockDefaultPositions();
            repositionBlocks();
            blockPathWidth = 12;
            blockRoundness = 8;
        } else if (window.innerWidth > 500 && maxGridWidth != 400) {
            maxGridWidth = 400;
            verticalBlockPadding = 7;
            horizontalBlockPadding = 2;
            await tick();
            await tick();
            await tick();
            await tick();
            setBlockDefaultPositions();
            repositionBlocks();
            blockPathWidth = 16;
            blockRoundness = 12;
        }
    }

    let gameDoneState = $state(false);
    let gameDoneSpring = new Spring(1, {
        stiffness: 0.15,
        damping: 0.22,
        precision: 0.0001,
    });

    let completePath: SVGPathElement | null = $state(null);
    let pathLength = new Tween(0, {
        duration: 1600,
        easing: sineInOut,
    });
    let disableInput = $state(false);
    function gameCompleteEffect() {
      disableInput = true;
        setTimeout(() => {
            if (completePath == null) return;
            let length = completePath?.getTotalLength();
            completePath.style.opacity = "1";
            completePath.style.strokeDasharray = `${length}`;
            pathLength.set(length, { duration: 0 });
            setTimeout(() => {
                pathLength.set(0);
            }, 10);
            setTimeout(() => {
                blockRoundness = 6;
                gameDoneState = true;
                verticalBlockPadding = 0;
                horizontalBlockPadding = 0;
                gameDoneSpring.set(1.1);
                setTimeout(() => {
                    gameDoneSpring.set(1, {
                        preserveMomentum: 1,
                    });
                }, 100);
            }, 2000);
        }, 1000);
    }
</script>

<svelte:window onresize={onResize} />

<main>
    <div class="game-wrapper">
        <div class="top-details">
            <div class="word-container" id="word-container"></div>
            <div class="points-bar"></div>
        </div>
        <div class="game-zone">
            <div
                class="game-grid"
                bind:this={gameGrid}
                style:width={`${gridBlockWidth * gridX}px`}
            >
                <div class="game-actual-grid">
                    {#each Array(gridY) as _, y}
                        {#each Array(gridX) as _, x}
                            {#key blocks}
                                <div
                                    class="game-grid-cell"
                                    style:grid-column={x + 1}
                                    style:grid-row={y + 1}
                                    style:width="{gridBlockWidth}px"
                                    style:height="{gridBlockHeight}px"
                                    style:border-radius={`${blockRoundness}px`}
                                ></div>{/key}
                        {/each}
                    {/each}
                </div>
                <div
                    class="game-block-selectors"
                    bind:this={gameBlockSelectors}
                >
                    {#each blocks as block, i (block.id)}
                        {@const bounds = boundsOfArray([
                            ...block.parts.map((p) => ({
                                x: block.truthPosition.x + p.x,
                                y: block.truthPosition.y + p.y,
                            })),
                        ])}
                        {#if !block.locked}
                            <div
                                class="block-selector-zone"
                                bind:this={blocks[i].blockHome}
                                style:pointer-events="none"
                            >
                                {#if !block.placed}
                                    <div
                                        style:opacity={block.dragging ||
                                        block.isCurrentlyAnimating
                                            ? "0"
                                            : "1"}
                                        class="game-block-transform-wrapper"
                                        class:game-block-selectable={true}
                                        style:width={(bounds.absolute.maxX +
                                            1) *
                                            gridBlockWidth +
                                            "px"}
                                        style:height={(bounds.absolute.maxY +
                                            1) *
                                            gridBlockHeight +
                                            "px"}
                                        style:top={"50%"}
                                        style:--block-position-x={"-50%"}
                                        style:--block-position-y={"-50%"}
                                        style:left={"50%"}
                                        style:--shadow-color={block.color
                                            .darken(0.5)
                                            .hex() as any as string}
                                    >
                                        <div
                                            class:game-block-shadow-wrapper={!isWebKit}
                                        >
                                            {@render blockSVG(block)}
                                        </div>
                                    </div>
                                {/if}
                            </div>
                        {/if}
                    {/each}
                </div>
                <div
                    class="grid-wrapper-scale"
                    style:transform={`scale(${gameDoneSpring.current})`}
                    style:width={`${gridBlockWidth * gridX}px`}
                    style:height={`${gridBlockHeight * gridY}px`}
                >
                    {#each blocks as block}
                        {@const bounds = boundsOfArray([
                            ...block.parts.map((p) => ({
                                x: block.truthPosition.x + p.x,
                                y: block.truthPosition.y + p.y,
                            })),
                        ])}
                        <div
                            style:opacity={block.dragging ||
                            block.placed ||
                            block.isCurrentlyAnimating
                                ? "1"
                                : "0"}
                            class:ignore-all-events={!block.placed || disableInput}
                            class="game-block-transform-wrapper"
                            class:game-complete-block={gameDoneState}
                            class:game-block-selectable={!block.placed}
                            style:width={(bounds.absolute.maxX + 1) *
                                gridBlockWidth +
                                "px"}
                            style:height={(bounds.absolute.maxY + 1) *
                                gridBlockHeight +
                                "px"}
                            style:top={"0px"}
                            style:--block-position-x={block.cssPosition.current
                                .x + "px"}
                            style:--block-position-y={block.cssPosition.current
                                .y + "px"}
                            style:left={"0px"}
                            style:--shadow-color={block.color
                                .darken(0.5)
                                .hex() as any as string}
                            bind:this={block.blockEl}
                        >
                            <div class:game-block-shadow-wrapper={!isWebKit}>
                                {@render blockSVG(block)}
                            </div>
                        </div>
                    {/each}

                    <div
                        class="preview-mask-wrapper"
                        style:width={60 + gridX * gridBlockWidth + "px"}
                        style:height={60 + gridY * gridBlockHeight + "px"}
                    >
                        {#if currentlyHovering && selectedBlock != null}
                            {@const block = selectedBlock}
                            {@const bounds = boundsOfArray([
                                ...block.parts.map((p) => ({
                                    x: currentHoveringPlacement.x + p.x,
                                    y: currentHoveringPlacement.y + p.y,
                                })),
                            ])}
                            <div
                                class="game-block-drop-preview"
                                style:width={(bounds.absolute.maxX + 1) *
                                    gridBlockWidth +
                                    "px"}
                                style:height={(bounds.absolute.maxY + 1) *
                                    gridBlockHeight +
                                    "px"}
                                style:top={"0px"}
                                style:--block-position-x={currentHoveringPlacement.x *
                                    gridBlockWidth +
                                    30 +
                                    "px"}
                                style:--block-position-y={currentHoveringPlacement.y *
                                    gridBlockHeight +
                                    30 +
                                    "px"}
                                style:left={"0px"}
                            >
                                <svg
                                    width={(bounds.absolute.maxX + 1) *
                                        gridBlockWidth}
                                    height={(bounds.absolute.maxY + 1) *
                                        gridBlockHeight}
                                    viewBox={`0 0 ${(bounds.absolute.maxX + 1) * gridBlockWidth} ${
                                        (bounds.absolute.maxY + 1) *
                                        gridBlockHeight
                                    }`}
                                >
                                    <defs>
                                        <mask
                                            id={`block-mask-drop-preview`}
                                            maskUnits="userSpaceOnUse"
                                            x="0"
                                            y="0"
                                            width={(bounds.absolute.maxX + 1) *
                                                gridBlockWidth}
                                            height={(bounds.absolute.maxY + 1) *
                                                gridBlockHeight}
                                        >
                                            {#each block.cardinalTwoPairs as pair}
                                                <path
                                                    class="game-block-cell"
                                                    d={`M ${(pair.x1 + currentHoveringPlacement.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + currentHoveringPlacement.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + currentHoveringPlacement.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + currentHoveringPlacement.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                                                    stroke-width={gridBlockWidth -
                                                        previewHoverPadding * 2}
                                                    stroke={"white"}
                                                ></path>
                                            {/each}
                                            {#each block.parts as part}
                                                <rect
                                                    class="game-block-cell"
                                                    rx={blockRoundness}
                                                    ry={blockRoundness}
                                                    x={(currentHoveringPlacement.x +
                                                        part.x -
                                                        bounds.relative.minX) *
                                                        gridBlockWidth +
                                                        previewHoverPadding}
                                                    y={(currentHoveringPlacement.y +
                                                        part.y -
                                                        bounds.relative.minY) *
                                                        gridBlockHeight +
                                                        previewHoverPadding}
                                                    width={gridBlockWidth -
                                                        previewHoverPadding * 2}
                                                    height={gridBlockHeight -
                                                        previewHoverPadding * 2}
                                                    fill={"white"}
                                                ></rect>
                                            {/each}

                                            {#each block.cardinalTwoPairs as pair}
                                                <path
                                                    class="game-block-cell"
                                                    d={`M ${(pair.x1 + currentHoveringPlacement.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + currentHoveringPlacement.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + currentHoveringPlacement.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + currentHoveringPlacement.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                                                    stroke-width={gridBlockWidth -
                                                        previewOutlineWidth * 2}
                                                    stroke={"#333"}
                                                ></path>
                                            {/each}
                                            {#each block.parts as part}
                                                <rect
                                                    class="game-block-cell"
                                                    rx={blockRoundness -
                                                        previewOutlineWidth}
                                                    ry={blockRoundness -
                                                        previewOutlineWidth}
                                                    x={(currentHoveringPlacement.x +
                                                        part.x -
                                                        bounds.relative.minX) *
                                                        gridBlockWidth +
                                                        previewOutlineWidth}
                                                    y={(currentHoveringPlacement.y +
                                                        part.y -
                                                        bounds.relative.minY) *
                                                        gridBlockHeight +
                                                        previewOutlineWidth}
                                                    width={gridBlockWidth -
                                                        previewOutlineWidth * 2}
                                                    height={gridBlockHeight -
                                                        previewOutlineWidth * 2}
                                                    fill={"#333"}
                                                ></rect>
                                            {/each}
                                        </mask>
                                    </defs>
                                    <g>
                                        <g>
                                            <rect
                                                class="block-fill-preview"
                                                mask="url(#block-mask-drop-preview)"
                                                x="0"
                                                y="0"
                                                width={(bounds.absolute.maxX +
                                                    1) *
                                                    gridBlockWidth}
                                                height={(bounds.absolute.maxY +
                                                    1) *
                                                    gridBlockHeight}
                                                fill={currentPlacementInvalid
                                                    ? "var(--error)"
                                                    : "var(--success)"}
                                            ></rect>
                                        </g>
                                    </g>
                                </svg>
                            </div>
                        {/if}
                    </div>
                    <svg
                        class="path-overlay"
                        viewBox={`0 0 ${gridBlockWidth * gridX} ${gridBlockHeight * gridY}`}
                        style:transform={`translateY(-${gameDoneState ? 0 : 5}px)`}
                    >
                        <path
                            d={`M ${path[0].x * gridBlockWidth + gridBlockWidth / 2} ${path[0].y * gridBlockHeight + gridBlockHeight / 2} ` +
                                path
                                    .map(
                                        (p) =>
                                            `L ${p.x * gridBlockWidth + gridBlockWidth / 2} ${p.y * gridBlockHeight + gridBlockHeight / 2} `,
                                    )
                                    .join(" ")}
                            stroke-width="10px"
                            stroke="white"
                            stroke-linecap="round"
                            stroke-linejoin="round"
                            fill="transparent"
                            bind:this={completePath}
                            stroke-dashoffset={pathLength.current}
                        ></path>
                    </svg>
                </div>
            </div>
        </div>
    </div>
</main>

{#snippet blockSVG(block: Block)}
    {@const bounds = boundsOfArray([
        ...block.parts.map((p) => ({
            x: block.truthPosition.x + p.x,
            y: block.truthPosition.y + p.y,
        })),
    ])}
    <div
        class="game-block"
        class:block-dragging={block.dragging}
        style:animation-delay={block.animDelay}
        class:game-block-locked={block.locked}
        style:width={(bounds.absolute.maxX + 1) * gridBlockWidth + "px"}
        style:height={(bounds.absolute.maxY + 1) * gridBlockHeight + "px"}
        style:--shadow-color={block.color.darken(0.5).hex() as any as string}
        style:--sway-angle={block.dragging ? `${dragSway * 2}deg` : "0deg"}
        onpointerdown={(e) => {
            blockPointerDown(block, e);
        }}
        onpointermove={(e) => {
            pointerMove(e);
        }}
        onpointerup={(e) => {
            pointerUp(e);
        }}
        tabindex="-1"
        role="button"
    >
        <svg
            class="game-block-svg"
            width={(bounds.absolute.maxX + 1) * gridBlockWidth}
            height={(bounds.absolute.maxY + 1) * gridBlockHeight}
            viewBox={`0 0 ${(bounds.absolute.maxX + 1) * gridBlockWidth} ${
                (bounds.absolute.maxY + 1) * gridBlockHeight
            }`}
        >
            <defs>
                <mask
                    id={`block-mask-${block.id}`}
                    maskUnits="userSpaceOnUse"
                    x="0"
                    y="0"
                    width={(bounds.absolute.maxX + 1) * gridBlockWidth}
                    height={(bounds.absolute.maxY + 1) * gridBlockHeight}
                >
                    <g>
                        {#each block.cardinalTwoPairs as pair}
                            <path
                                class="game-block-cell"
                                d={`M ${(pair.x1 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                                stroke-width={pair.x1 == pair.x2
                                    ? gridBlockWidth -
                                      horizontalBlockPadding * 2
                                    : gridBlockHeight -
                                      verticalBlockPadding * 2}
                                stroke="white"
                                fill="transparent"
                            ></path>
                        {/each}
                        {#each block.parts as part}
                            <rect
                                class="game-block-cell"
                                rx={blockRoundness}
                                ry={blockRoundness}
                                x={(block.truthPosition.x +
                                    part.x -
                                    bounds.relative.minX) *
                                    gridBlockWidth +
                                    horizontalBlockPadding}
                                y={(block.truthPosition.y +
                                    part.y -
                                    bounds.relative.minY) *
                                    gridBlockHeight +
                                    verticalBlockPadding}
                                width={gridBlockWidth -
                                    horizontalBlockPadding * 2}
                                height={gridBlockHeight -
                                    verticalBlockPadding * 2}
                                fill="white"
                            ></rect>
                        {/each}
                    </g>
                </mask>
            </defs>
            <g>
                <g>
                    {#each block.cardinalTwoPairs as pair}
                        <path
                            class="game-block-cell"
                            d={`M ${(pair.x1 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                            stroke-width={pair.x1 == pair.x2
                                ? gridBlockWidth - horizontalBlockPadding * 2
                                : gridBlockHeight - verticalBlockPadding * 2}
                            stroke={block.color.hex() as any as string}
                        ></path>
                    {/each}
                    {#each block.parts as part}
                        <rect
                            class="game-block-cell"
                            rx={blockRoundness}
                            ry={blockRoundness}
                            x={(block.truthPosition.x +
                                part.x -
                                bounds.relative.minX) *
                                gridBlockWidth +
                                horizontalBlockPadding}
                            y={(block.truthPosition.y +
                                part.y -
                                bounds.relative.minY) *
                                gridBlockHeight +
                                verticalBlockPadding}
                            width={gridBlockWidth - horizontalBlockPadding * 2}
                            height={gridBlockHeight - verticalBlockPadding * 2}
                            fill={block.color.hex() as any as string}
                        ></rect>
                    {/each}
                    {#each block.parts as part}
                        <rect
                            class="game-block-cell"
                            rx={blockRoundness - strokePadding}
                            ry={blockRoundness - strokePadding}
                            x={(block.truthPosition.x +
                                part.x -
                                bounds.relative.minX) *
                                gridBlockWidth +
                                strokePadding +
                                horizontalBlockPadding}
                            y={(block.truthPosition.y +
                                part.y -
                                bounds.relative.minY) *
                                gridBlockHeight +
                                strokePadding +
                                verticalBlockPadding}
                            width={gridBlockWidth -
                                horizontalBlockPadding * 2 -
                                strokePadding * 2}
                            height={gridBlockHeight -
                                verticalBlockPadding * 2 -
                                strokePadding * 2}
                            stroke={block.color
                                .brighten(0.4)
                                .hex() as any as string}
                            stroke-width="4"
                            fill="transparent"
                        ></rect>
                    {/each}

                    {#each block.cardinalTwoPairs as pair}
                        <path
                            class="game-block-cell"
                            d={`M ${(pair.x1 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                            stroke-width={(pair.x1 == pair.x2
                                ? gridBlockWidth - horizontalBlockPadding * 2
                                : gridBlockHeight - verticalBlockPadding * 2) -
                                (strokePadding - 2) * 2}
                            stroke={block.color
                                .brighten(0.4)
                                .hex() as any as string}
                        ></path>
                    {/each}
                    {#each block.cardinalTwoPairs as pair}
                        <path
                            class="game-block-cell"
                            d={`M ${(pair.x1 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.truthPosition.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.truthPosition.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
                            stroke-width={(pair.x1 == pair.x2
                                ? gridBlockWidth - horizontalBlockPadding * 2
                                : gridBlockHeight - verticalBlockPadding * 2) -
                                (strokePadding + 2) * 2}
                            stroke={block.color.hex() as any as string}
                        ></path>
                    {/each}
                </g>
                <path
                    mask={`url(#block-mask-${block.id})`}
                    d={`M ${(path[0].x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(path[0].y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} ` +
                        path
                            .map(
                                (p) =>
                                    `L ${(p.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(p.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} `,
                            )
                            .join(" ")}
                    stroke-width={blockPathWidth + 8}
                    stroke={block.color.brighten(1.2).hex()}
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    fill="transparent"
                ></path>
                <path
                    mask={`url(#block-mask-${block.id})`}
                    d={`M ${(path[0].x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(path[0].y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} ` +
                        path
                            .map(
                                (p) =>
                                    `L ${(p.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(p.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} `,
                            )
                            .join(" ")}
                    stroke-width={blockPathWidth}
                    stroke={block.color.darken(1.8).hex()}
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    fill="transparent"
                    stroke-dashoffset={pathLength.current}
                ></path>
            </g>
        </svg>
    </div>
{/snippet}

<style>
    .game-wrapper {
        display: flex;
        flex-direction: column;
        height: 100%;
    }
    .game-zone {
        flex-grow: 1;
    }
    .game-grid {
        flex-grow: 1;
    }
    
    .game-splash-inner {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;

        opacity: 0;
        transition: opacity 0.3s;
    }
    .game-splash-ready {
        opacity: 1;
    }
    .game-splash-wrapper {
        background: #73bf9c;
        z-index: 9999999999999999 !important;
    }
    .grid-wrapper-scale {
        position: absolute;
        top: 0;
        left: 0;
        z-index: 999999999;
    }
    .game-zone {
        display: flex;
        flex-direction: column;
        align-items: center;
    }
    .game-grid {
        position: relative;
        display: flex;
        flex-direction: column;
        box-sizing: border-box;
    }
    .game-actual-grid {
        display: grid;
    }
    .path-overlay {
        width: 100%;
        height: 100%;
        position: absolute;
        top: 0;
        left: 0;
        z-index: 99;
        z-index: 999999;
        pointer-events: none;
        opacity: 1;
    }
    .path-overlay path {
        opacity: 0;
    }

    .block-selector-zone {
        width: 70px;
        height: 70px;
        background: var(--raised-surface);
        border-radius: 100%;
        position: relative;
        z-index: 1;
        pointer-events: none;
    }
    .game-block-selectors {
        width: 100%;
        height: 100%;
        gap: 8px;
        margin-top: 0px;
        padding: 30px 0px;
        overflow-x: auto;
        overflow-y: hidden;

        box-sizing: border-box;
        display: grid;
        grid-template-rows: repeat(auto-fill, 70px);
        grid-auto-flow: column;
        grid-auto-columns: 1fr;
        max-height: 230px;
    }
    .game-grid-cell {
        background: var(--raised-surface);
        border: 2px solid var(--body-background);
        box-sizing: border-box;
    }
    .game-grid-cell.source {
        outline: 4px solid blue;
        z-index: 9999;
    }
    .game-grid-cell.sink {
        outline: 4px solid red;
        z-index: 9999;
    }
    .game-block-cell {
        cursor: grab;

        touch-action: none !important;
        pointer-events: all;
    }
    .game-block-locked .game-block-cell {
        cursor: not-allowed !important;
    }
    .game-block {
        --sway-angle: 0deg;
        --scale-mult: 1;
        --scale-hover-mult: 1;
        --sway-duration: 2s;
        position: absolute;
        top: 0;
        left: 0;
        transition: transform 0.4s cubic-bezier(0.18, 1.46, 0.65, 0.98);
        pointer-events: none;
        transform: scale(calc(var(--scale-hover-mult) * var(--scale-mult)))
            rotate(var(--sway-angle));
        touch-action: none;
    }
    .game-block-transform-wrapper {
        --block-position-x: 0px;
        --block-position-y: 0px;
        position: absolute;
        pointer-events: none;
        transform: translate(
            var(--block-position-x),
            calc(var(--block-position-y) - 5px)
        );
    }
    .game-block-shadow-wrapper {
        transition: filter 0.2s;
        filter: drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color));
    }
    .game-block-drop-preview {
        position: absolute;
        pointer-events: none !important;
        transform: translate(
            var(--block-position-x),
            calc(var(--block-position-y))
        );
        transition: transform 0.1s;
    }
    .game-block:has(.game-block-cell:hover):not(.game-block-locked) {
        --scale-hover-mult: 1.02;
        z-index: 99999;
    }
    .game-block-transform-wrapper:has(.game-block-cell:hover) {
        z-index: 99999;
    }
    .game-block-transform-wrapper.game-block-selectable {
        transform: translate(
            var(--block-position-x),
            calc(var(--block-position-y) - 5px)
        );
    }
    .game-block-transform-wrapper.game-block-selectable
        .game-block-shadow-wrapper {
        filter: drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color));
    }
    .game-block-selectable .game-block {
        --scale-mult: 0.4;
        animation: sway var(--sway-duration) infinite both alternate-reverse
            ease-in-out;
    }
    .game-block-transform-wrapper:has(.block-dragging) {
        z-index: 9999999999999 !important;
    }
    .block-dragging * {
        z-index: 9999999999999 !important;
    }
    .game-block-transform-wrapper:has(.block-dragging)
        .game-block-shadow-wrapper {
        filter: drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color))
            drop-shadow(0px 1px 0px var(--shadow-color))
            drop-shadow(0px 2px 0px var(--shadow-color));
    }
    .game-block.block-dragging {
        --scale-mult: 0.7;
        animation: unset !important;
        z-index: 9999999999999;
    }
    .game-block-svg {
        position: absolute;
        top: 0;
        left: 0;
        pointer-events: none;
        z-index: 99999;
        transition: filter 0.2s;
        filter: drop-shadow(0px 10px 0px var(--shadow-color));
    }
    .game-block-svg * {
        transition: 0.2s;
    }
    @keyframes sway {
        0% {
            --sway-angle: -3deg;
        }
        100% {
            --sway-angle: 3deg;
        }
    }
    .block-fill-preview {
        transition: fill 0.1s;
    }
    .preview-mask-wrapper {
        pointer-events: none !important;
        position: absolute;
        z-index: 99999999;
        top: -30px;
        left: -30px;
        overflow: hidden;
        mask-mode: luminance;
        mask-composite: intersect;
        mask-image:
            linear-gradient(
                to right,
                black,
                white 40px,
                white calc(100% - 40px),
                black
            ),
            linear-gradient(
                to bottom,
                black,
                white 40px,
                white calc(100% - 40px),
                black
            );
        -webkit-mask-mode: luminance;
        -webkit-mask-composite: intersect;
        -webkit-mask-image:
            linear-gradient(
                to right,
                black,
                white 40px,
                white calc(100% - 40px),
                black
            ),
            linear-gradient(
                to bottom,
                black,
                white 40px,
                white calc(100% - 40px),
                black
            );
    }
    @media (max-width: 500px) {
        .game-block-selectable .game-block {
            --scale-mult: 0.55;
        }

        .game-block.block-dragging {
            --scale-mult: 0.8;
        }
        .block-selector-zone {
            width: 80px;
            height: 80px;
        }
        .game-block-selectors {
            
            grid-template-rows: repeat(auto-fill, 80px);
        }
    }

    .ignore-all-events *,
    .ignore-all-events {
        pointer-events: none !important;
    }
    .game-complete-block {
        filter: none;
        transform: translate(
            var(--block-position-x),
            calc(var(--block-position-y))
        );
        pointer-events: none;
    }
    .game-complete-block .game-block-shadow-wrapper {
        filter: none;
    }
    .game-complete-block .game-block-svg {
        filter: none;
    }
    .game-block-shadow-wrapper .game-block-svg {
        filter: none;
    }
    .game-complete-block * {
        pointer-events: none !important;
    }
</style>
