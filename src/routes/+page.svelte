<script lang="ts">
	import { onMount } from "svelte";
	import chroma, { type Color } from "chroma-js";
	import DTLogo from "$lib/assets/dt-logo.png";
	import { DTGameCore } from "$lib/dailytrojan-lib/gameCore";

	let DTGCore: DTGameCore;

	let gridX = 6;
	let gridY = 6;
	let maxGridWidth = 400;
	let maxGridHeight = 420;
	let gridBlockWidth = Math.ceil(maxGridWidth / gridX);
	let gridBlockHeight = Math.ceil(maxGridHeight / gridY);
	let source = $state({ x: 0, y: 0 });
	let sink = $state({ x: gridX - 1, y: gridY - 1 });

	let maxWaypoints = 2;
	let waypoints: { x: number; y: number }[] = $state([]);
	let allPositions: { x: number; y: number }[] = [];

	let path: { x: number; y: number }[] = $state([{ x: -10, y: -10 }]);
	let difficulties = [
		{ gridX: 6, gridY: 6, waypoints: 2 },
		{ gridX: 6, gridY: 6, waypoints: 3 },
	];

	let blocks: {
		position: { x: number; y: number };
		parts: { x: number; y: number }[];
		color: Color;
		id: string;
		cardinalTwoPairs: { x1: number; y1: number; x2: number; y2: number }[]; // pairs of cardinal parts that are both in the block, used for validation
		signalPath: { x: number; y: number }[]; //the part of the signal path that is contained in this block, used for validation
	}[] = $state([]);

	function genRandomId() {
		return DTGCore.randomFloat().toString(36).substr(2, 9);
	}

	function generateCorrectPath() {
		source.x = Math.floor(DTGCore.randomFloat() * (gridX * 0.3));
		source.y = Math.floor(DTGCore.randomFloat() * (gridY * 0.3));
		sink.x =
			Math.floor(DTGCore.randomFloat() * (gridX * 0.3)) + Math.floor(gridX * 0.7);
		sink.y =
			Math.floor(DTGCore.randomFloat() * (gridY * 0.3)) + Math.floor(gridY * 0.7);
		waypoints = [];
		allPositions = [];
		for (let x = 0; x < gridX; x++) {
			for (let y = 0; y < gridY; y++) {
				allPositions.push({ x, y });
			}
		}
		let possibleWaypoints = [...allPositions];
		possibleWaypoints.splice(
			possibleWaypoints.findIndex((p) => p.x === source.x && p.y === source.y),
			1,
		);
		possibleWaypoints.splice(
			possibleWaypoints.findIndex((p) => p.x === sink.x && p.y === sink.y),
			1,
		);
		//remove positions that lie on edges bc those can get messy
		for (let i = 0; i < gridX; i++) {
			let idxTop = possibleWaypoints.findIndex((p) => p.x == i && p.y == 0);
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
			let idxTop = possibleWaypoints.findIndex((p) => p.x == 0 && p.y == i);
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
			let idx = Math.floor(DTGCore.randomFloat() * possibleWaypoints.length);
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
				let found = visited.findIndex((p) => p.x == tP.x && p.y == tP.y);

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
							Math.pow(currentPosition.y + d.y - nextWaypoint.y, 2),
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

		return path.findIndex((p) => p.x == sink.x && p.y == sink.y) != -1;
	}

	function generateBlocks() {
		blocks = [];
		let occupied: { x: number; y: number }[] = [
			{ x: 0, y: 0 },
			{ x: 0, y: gridY - 1 },
			{ x: gridX - 1, y: gridY - 1 },
			{ x: gridX - 1, y: 0 },
		];
		let amountOfBlocks = Math.ceil((gridX * gridY) / 4);
		let colors = generateBeautifulColors(amountOfBlocks, {
			hueStep: 15,
			hueJitter: 12,
			lightRange: [0.2, 0.8],
			satRange: [0.2, 0.8],
		});
		//add the four corners to the blocks
		blocks.push({
			position: { x: 0, y: 0 },
			parts: [],
			color: colors[0],
			signalPath: [],
			cardinalTwoPairs: [],
			id: genRandomId(),
		});
		blocks.push({
			position: { x: 0, y: gridY - 1 },
			parts: [],
			color: colors[1],
			signalPath: [],
			cardinalTwoPairs: [],
			id: genRandomId(),
		});
		blocks.push({
			position: { x: gridX - 1, y: gridY - 1 },
			parts: [],
			color: colors[2],
			signalPath: [],
			cardinalTwoPairs: [],
			id: genRandomId(),
		});
		blocks.push({
			position: { x: gridX - 1, y: 0 },
			parts: [],
			color: colors[3],
			signalPath: [],
			cardinalTwoPairs: [],
			id: genRandomId(),
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
		console.log(potentialGrowthPoints);
		for (let i = 0; i < amountOfBlocks - 4; i++) {
			let p =
				potentialGrowthPoints[
					Math.floor(DTGCore.randomFloat() * potentialGrowthPoints.length)
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
				parts: [],
				color: colors[i + 4],
				signalPath: [],
				cardinalTwoPairs: [],
				id: genRandomId(),
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
				let found = occupied.findIndex((p) => p.x == tP.x && p.y == tP.y);

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
				let validDirectionsFromOrigin = getValidDirections(block.position);
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
					validPositions[Math.floor(DTGCore.randomFloat() * validPositions.length)];
				console.log(pos.x, block.position.x);
				block.parts.push({
					x: pos.x - block.position.x,
					y: pos.y - block.position.y,
				});
				occupied.push({ ...pos });
			}
		}
		//now we need to go through each part of the blcok, including the origin, and for each of them see if there is any cardinal pair, that being a cell of this block that is either up down left or right

		blocks.forEach((b) => {
			let cardinalTwoPairs: {
				x1: number;
				y1: number;
				x2: number;
				y2: number;
			}[] = [];
			let allParts = [{ x: 0, y: 0 }, ...b.parts];
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
		console.log($state.snapshot(blocks));
	}

	onMount(() => {
		DTGCore = new DTGameCore(null, null);
		window.DTGCore = DTGCore;
		generateCorrectPath();
		generateBlocks();
	});

	function stresstest() {
		let fails = 0;
		let i = 0;
		let intrv = setInterval(() => {
			let s = generateCorrectPath();
			if (!s) {
				fails++;
			}
			i++;
			if (i == 100) {
				clearInterval(intrv);
				console.log(fails);
			}
		}, 0);
	}
	let verticalBlockPadding = 7;
	let horizontalBlockPadding = 2;
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
		console.log(minX, maxX, minY, maxY);
		//also return bounds with min at 0, adjusting max accordingly
		return {
			relative: { minX, maxX, minY, maxY },
			absolute: { minX: 0, maxX: maxX - minX, minY: 0, maxY: maxY - minY },
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
		const rand = (a: number, b: number) => a + DTGCore.randomFloat() * (b - a);

		const baseHue = rand(0, 360);
		const step = 360 / count;

		const colors = [];

		for (let i = 0; i < count; i++) {
			const hue =
				(baseHue + i * hueStep + rand(-hueJitter, hueJitter) + 360) % 360;
			const sat = clamp(
				rand(satRange[0], satRange[1]) + rand(-satJitter, satJitter),
				0,
				1,
			);
			const light = clamp(
				rand(lightRange[0], lightRange[1]) + rand(-lightJitter, lightJitter),
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
	function validateWin() {}

	let strokePadding = 7;
	let blockRoundness = 10;
</script>

<header>
	<img
		id="dt-logo"
		src={DTLogo}
		height="50"
		width="auto"
		style="cursor: pointer;"
		alt="Daily Trojan Logo"
	/>
</header>

<main>
	<div class="game-wrapper">
		<h1>Signals</h1>
		<div class="top-details">
			<div class="word-container" id="word-container"></div>
			<div class="points-bar"></div>
		</div>
		<div class="game-zone">
			<div
				class="game-grid"
				style:width={`${gridBlockWidth * gridX}px`}
				style:height={`${gridBlockHeight * gridY}px`}
			>
				<svg
					class="path-overlay"
					viewBox={`0 0 ${gridBlockWidth * gridX} ${gridBlockHeight * gridY}`}
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
					></path>
				</svg>
				{#each blocks as block}
					{@const bounds = boundsOfArray([
						block.position,
						...block.parts.map((p) => ({
							x: block.position.x + p.x,
							y: block.position.y + p.y,
						})),
					])}
					<div
						class="game-block-shadow-wrapper"
						style:width={(bounds.absolute.maxX + 1) * gridBlockWidth + "px"}
						style:height={(bounds.absolute.maxY + 1) * gridBlockHeight + "px"}
						style:top={bounds.relative.minY * gridBlockHeight + "px"}
						style:--shadow-color={block.color
							.darken(0.5)
							.hex() as any as string}
						style:left={bounds.relative.minX * gridBlockWidth + "px"}
					>
						<div
							class="game-block"
							style:width={(bounds.absolute.maxX + 1) * gridBlockWidth -
								horizontalBlockPadding * 2 +
								"px"}
							style:height={(bounds.absolute.maxY + 1) * gridBlockHeight -
								verticalBlockPadding * 2 +
								"px"}
							style:--shadow-color={block.color
								.darken(0.5)
								.hex() as any as string}
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
													d={`M ${(pair.x1 + block.position.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.position.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.position.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.position.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
													stroke-width={pair.x1 == pair.x2
														? gridBlockWidth - horizontalBlockPadding * 2
														: gridBlockHeight - verticalBlockPadding * 2}
													stroke="white"
													fill="transparent"
												></path>
											{/each}
											<rect
												class="game-block-cell"
												rx="8"
												ry="8"
												x={(block.position.x - bounds.relative.minX) *
													gridBlockWidth +
													horizontalBlockPadding}
												y={(block.position.y - bounds.relative.minY) *
													gridBlockHeight +
													verticalBlockPadding}
												width={gridBlockWidth - horizontalBlockPadding * 2}
												height={gridBlockHeight - verticalBlockPadding * 2}
												fill="white"
											></rect>
											{#each block.parts as part}
												<rect
													class="game-block-cell"
													rx="8"
													ry="8"
													x={(block.position.x +
														part.x -
														bounds.relative.minX) *
														gridBlockWidth +
														horizontalBlockPadding}
													y={(block.position.y +
														part.y -
														bounds.relative.minY) *
														gridBlockHeight +
														verticalBlockPadding}
													width={gridBlockWidth - horizontalBlockPadding * 2}
													height={gridBlockHeight - verticalBlockPadding * 2}
													fill="white"
												></rect>
											{/each}
										</g>
									</mask>
								</defs>
								<g>
									{#each block.cardinalTwoPairs as pair}
										<path
											class="game-block-cell"
											d={`M ${(pair.x1 + block.position.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y1 + block.position.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} L ${(pair.x2 + block.position.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(pair.y2 + block.position.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2}`}
											stroke-width={pair.x1 == pair.x2
												? gridBlockWidth - horizontalBlockPadding * 2
												: gridBlockHeight - verticalBlockPadding * 2}
											stroke={block.color.hex() as any as string}
										></path>
									{/each}
									<rect
										class="game-block-cell"
										rx={blockRoundness}
										ry={blockRoundness}
										x={(block.position.x - bounds.relative.minX) *
											gridBlockWidth +
											horizontalBlockPadding}
										y={(block.position.y - bounds.relative.minY) *
											gridBlockHeight +
											verticalBlockPadding}
										width={gridBlockWidth - horizontalBlockPadding * 2}
										height={gridBlockHeight - verticalBlockPadding * 2}
										fill={block.color.hex() as any as string}
									></rect>
									{#each block.parts as part}
										<rect
											class="game-block-cell"
											rx={blockRoundness}
											ry={blockRoundness}
											x={(block.position.x + part.x - bounds.relative.minX) *
												gridBlockWidth +
												horizontalBlockPadding}
											y={(block.position.y + part.y - bounds.relative.minY) *
												gridBlockHeight +
												verticalBlockPadding}
											width={gridBlockWidth - horizontalBlockPadding * 2}
											height={gridBlockHeight - verticalBlockPadding * 2}
											fill={block.color.hex() as any as string}
										></rect>
									{/each}
									<path
										mask={`url(#block-mask-${block.id})`}
										d={`M ${(path[0].x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(path[0].y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} ` +
											path
												.map(
													(p) =>
														`L ${(p.x - bounds.relative.minX) * gridBlockWidth + gridBlockWidth / 2} ${(p.y - bounds.relative.minY) * gridBlockHeight + gridBlockHeight / 2} `,
												)
												.join(" ")}
										stroke-width="30px"
										stroke={block.color.darken(0.2).hex()}
										stroke-linecap="round"
										stroke-linejoin="round"
										fill="transparent"
									></path>
									<rect
										class="game-block-cell"
										rx={blockRoundness - strokePadding}
										ry={blockRoundness - strokePadding}
										x={(block.position.x - bounds.relative.minX) *
											gridBlockWidth +
											strokePadding +
											horizontalBlockPadding}
										y={(block.position.y - bounds.relative.minY) *
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
											.brighten(1)
											.alpha(0.2)
											.hex() as any as string}
										stroke-width="4"
										fill="transparent"
									></rect>
									{#each block.parts as part}
										<rect
											class="game-block-cell"
											rx={blockRoundness - strokePadding}
											ry={blockRoundness - strokePadding}
											x={(block.position.x + part.x - bounds.relative.minX) *
												gridBlockWidth +
												strokePadding +
												horizontalBlockPadding}
											y={(block.position.y + part.y - bounds.relative.minY) *
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
												.brighten(1)
												.alpha(0.2)
												.hex() as any as string}
											stroke-width="4"
											fill="transparent"
										></rect>
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
									stroke-width="20px"
									stroke={block.color.darken(1.5).hex()}
									stroke-linecap="round"
									stroke-linejoin="round"
									fill="transparent"
								></path>
							</svg>
						</div>
					</div>
				{/each}
				{#each Array(gridY) as _, y}
					{#each Array(gridX) as _, x}
						{#key blocks}
							<div
								class="game-grid-cell"
								style:grid-column={x + 1}
								style:grid-row={y + 1}
								style:width="{gridBlockWidth}px"
								style:height="{gridBlockHeight}px"
							></div>{/key}
					{/each}
				{/each}
			</div>
			<button onclick={stresstest}>stress test</button>
			<button onclick={generateCorrectPath}>generateCorrectPath</button>
			<button onclick={generateBlocks}>generateBlocks</button>
		</div>
	</div>
</main>

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
		z-index: 999999;
		pointer-events: none;
		transform: translateY(-6px);
		opacity: 0;
	}
	.game-grid-cell {
		background: #e3e3e3;
		border-radius: 18px;
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
		pointer-events: all !important;
	}
	.game-block {
		position: absolute;
		top: 0;
		left: 0;
		transition: transform 0.2s;
		pointer-events: none;
	}
	.game-block-shadow-wrapper {
		position: absolute;
		filter: drop-shadow(0px 2px 0px var(--shadow-color))
			drop-shadow(0px 2px 0px var(--shadow-color))
			drop-shadow(0px 2px 0px var(--shadow-color))
			drop-shadow(0px 2px 0px var(--shadow-color))
			drop-shadow(0px 2px 0px var(--shadow-color));
		pointer-events: none;
		transform: translateY(-6px);
	}
	.game-block:has(.game-block-cell:hover) {
		transform: scale(1.02);
		z-index: 99999;
	}
	.game-block-shadow-wrapper:has(.game-block-cell:hover) {
		z-index: 99999;
	}
	.hitbox {
		position: absolute;
		left: -2px;
		top: -6px;
		bottom: -6px;
		right: -2px;
		background: transparent;
	}
	.game-block-svg {
		position: absolute;
		top: 0;
		left: 0;
		pointer-events: none;
		z-index: 99999;
	}
</style>
