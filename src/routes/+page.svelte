<script lang="ts">
    import SignalsLogo from "$lib/assets/signals.svg";
    import { goto } from "$app/navigation";
    let difficulty = $state("easy")
</script>

<div class="game-splash-wrapper" id="splash">
    <div class="game-splash-inner" class:game-splash-ready={true}>
        <img width="80" src={SignalsLogo} alt="Signals Logo" />
        <h1>Signals</h1>
        <h2 id="game-tagline">
            Connect the signal by placing the blocks in order. 
        </h2>
        <div class="difficulty-selector">
            <button class="difficulty-option" onclick={()=>{
              difficulty = "easy"
            }} class:active={difficulty == "easy"}>Easy</button>
            <button class="difficulty-option" onclick={()=>{
              difficulty = "medium"
            }}  class:active={difficulty == "medium"}>Medium</button>
            <button class="difficulty-option" onclick={()=>{
              difficulty = "hard"
            }}  class:active={difficulty == "hard"}>Hard</button>
        </div>
        <div class="flex-hor">
            <button id="game-back-button">Back</button>
            <button id="game-action-button" onclick={()=>{goto("/" + difficulty)}}>Play</button>
        </div>
        <p id="splash-date">date</p>
        <p id="splash-date">Game by Noah Pinales</p>
    </div>
</div>

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
    }
    .difficulty-option:hover {
        color: black !important;
        background: initial
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