<script lang="ts">
    import SignalsLogo from "$lib/assets/signals.svg";
    import { goto } from "$app/navigation";
    let difficulty: "easy" | "medium" | "hard" = $state("easy");
    import Game from "$lib/components/Game.svelte";
    import { onMount } from "svelte";
    let gameStart = $state(false);
    let gameComponent: Game | null = $state(null)
    let complete = $state([false, false, false])
    
    let gameSeed = new Intl.DateTimeFormat("en-US", {
        year: "numeric",
        month: "2-digit",
        day: "2-digit",
    }).format(new Date());

    function playGame() {
      if(gameComponent == null ) return
      gameStart = true;
      gameComponent.playGame();
    }
    
    function loadData(id: string) {
        let item = localStorage.getItem(id);
        if (item != null) return JSON.parse(item);
        else return null;
    }
    
    function updateSplash() {
      let easyData = loadData('signals-today-easy')
      let mediumData = loadData('signals-today-medium')
      let hardData = loadData('signals-today-hard')
      if(easyData != null && easyData.date == gameSeed) {
        complete[0] = easyData.over
      }
      if(mediumData != null && mediumData.date == gameSeed) {
        complete[1] = mediumData.over
      }
      if(hardData != null && hardData.date == gameSeed) {
        complete[2] = hardData.over
      }
      console.log(complete)
      
    }
    
    onMount(()=>{
      updateSplash();
    })
</script>

<div
    class="game-splash-wrapper"
    id="splash"
    class:game-splash-hidden={gameStart}
>
    <div class="game-splash-inner" class:game-splash-ready={true}>
        <img width="80" src={SignalsLogo} alt="Signals Logo" />
        <h1>Signals</h1>
        <h2 id="game-tagline">
            Connect the signal by placing the blocks in order.
        </h2>
        <div class="difficulty-selector">
            <button
                class="difficulty-option"
                onclick={() => {
                    difficulty = "easy";
                }}
                class:active={difficulty == "easy"}>
                    {#if complete[0]}
                        <i class="ti ti-check"></i>
                        {/if}
                    Easy</button
            >
            <button
                class="difficulty-option"
                onclick={() => {
                    difficulty = "medium";
                }}
                class:active={difficulty == "medium"}>
                    {#if complete[1]}
                        <i class="ti ti-check"></i>
                        {/if}Medium</button
            >
            <button
                class="difficulty-option"
                onclick={() => {
                    difficulty = "hard";
                }}
                class:active={difficulty == "hard"}>
                    {#if complete[2]}
                        <i class="ti ti-check"></i>
                        {/if}Hard</button
            >
        </div>
        <div class="flex-hor">
            <button id="game-back-button">Back</button>
            <button
                id="game-action-button"
                onclick={playGame}>Play</button
            >
        </div>
        <p id="splash-date">{new Intl.DateTimeFormat("en-US", {
          month: "long",
          day: "2-digit",
          year: "numeric",
        }).format(new Date())}</p>
        <p id="splash-date">Game by Noah Pinales</p>
    </div>
</div>


{#key difficulty}
<Game {difficulty} bind:this={gameComponent}></Game>
{/key}
<style>
    .difficulty-selector {
        display: flex;
        background: #44a587;
        margin-bottom: 20px;
        border-radius: 14px;
        padding: 4px;
        gap: 2px;
    }
    .difficulty-option {
        background: transparent;
        border: none;
        margin: 0 !important;
        gap: 4px;
        padding-left: 8px;
    }
    .difficulty-option:hover {
        color: black !important;
        background: initial;
    }
    .difficulty-option.active {
        background: white;
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
</style>
