<script lang="ts">
  import { onMount } from 'svelte';
  import { Coins, RotateCw, Trophy, History, Star, Volume2, VolumeX } from 'lucide-svelte';
  import { fade, fly, scale } from 'svelte/transition';
  import { tweened } from 'svelte/motion';
  import { cubicOut, elasticOut } from 'svelte/easing';

  // Game Constants
  const SYMBOLS = [
    { id: 'cherry', icon: '🍒', value: 10, color: 'text-red-500' },
    { id: 'lemon', icon: '🍋', value: 20, color: 'text-yellow-400' },
    { id: 'orange', icon: '🍊', value: 30, color: 'text-orange-500' },
    { id: 'plum', icon: '🍇', value: 50, color: 'text-purple-500' },
    { id: 'bell', icon: '🔔', value: 100, color: 'text-yellow-600' },
    { id: 'diamond', icon: '💎', value: 250, color: 'text-blue-400' },
    { id: 'seven', icon: '7️⃣', value: 500, color: 'text-red-600' },
  ];

  const REEL_COUNT = 3;
  const ROWS = 3;
  const RECOVERY_AMOUNT = 100;
  const RECOVERY_INTERVAL = 3600000; // 1 hour in ms

  // Sound Effects
  const SOUNDS = {
    spin: 'https://assets.mixkit.co/active_storage/sfx/2013/2013-preview.mp3',
    stop: 'https://assets.mixkit.co/active_storage/sfx/2571/2571-preview.mp3',
    win: 'https://assets.mixkit.co/active_storage/sfx/1435/1435-preview.mp3'
  };

  // Game State
  let balance = $state(1000);
  let isMuted = $state(false);
  let nextRecoveryTime = $state<number | null>(null);
  let timeUntilRecovery = $state<string>('');

  const displayedBalance = tweened(1000, {
    duration: 1000,
    easing: cubicOut
  });

  // Sync tweened balance
  $effect(() => {
    displayedBalance.set(balance);
  });

  // Persistence and Recovery Logic
  onMount(() => {
    const savedBalance = localStorage.getItem('slot_balance');
    const savedRecovery = localStorage.getItem('slot_next_recovery');
    
    if (savedBalance !== null) balance = parseInt(savedBalance);
    if (savedRecovery !== null) nextRecoveryTime = parseInt(savedRecovery);

    const interval = setInterval(checkRecovery, 1000);
    return () => clearInterval(interval);
  });

  $effect(() => {
    localStorage.setItem('slot_balance', balance.toString());
    if (balance === 0 && !nextRecoveryTime) {
      nextRecoveryTime = Date.now() + RECOVERY_INTERVAL;
      localStorage.setItem('slot_next_recovery', nextRecoveryTime.toString());
    } else if (balance > 0 && nextRecoveryTime) {
      nextRecoveryTime = null;
      localStorage.removeItem('slot_next_recovery');
    }
  });

  function checkRecovery() {
    if (balance > 0 || !nextRecoveryTime) return;

    const now = Date.now();
    const diff = nextRecoveryTime - now;

    if (diff <= 0) {
      balance = RECOVERY_AMOUNT;
      nextRecoveryTime = null;
      localStorage.removeItem('slot_next_recovery');
      timeUntilRecovery = '';
    } else {
      const minutes = Math.floor(diff / 60000);
      const seconds = Math.floor((diff % 60000) / 1000);
      timeUntilRecovery = `${minutes}:${seconds.toString().padStart(2, '0')}`;
    }
  }

  let betMultiplierIndex = $state(0);
  const BET_MULTIPLIERS = [1, 2, 5, 10, 20, 50, 100, 200, 500, 1000];
  const BASE_BET = 10;
  let bet = $derived(BASE_BET * BET_MULTIPLIERS[betMultiplierIndex]);

  let spinning = $state(false);
  let lastWin = $state(0);
  let reels = $state(Array(REEL_COUNT).fill(0).map(() => getRandomSymbols()));
  let history = $state<{ result: string, win: number, time: string }[]>([]);
  let showHistory = $state(false);
  let winParticles = $state<{ id: number, x: number, y: number }[]>([]);

  // Audio Objects
  let audioSpin: HTMLAudioElement;
  let audioStop: HTMLAudioElement;
  let audioWin: HTMLAudioElement;

  onMount(() => {
    audioSpin = new Audio(SOUNDS.spin);
    audioSpin.loop = true;
    audioSpin.volume = 0.2;
    
    audioStop = new Audio(SOUNDS.stop);
    audioStop.volume = 0.3;
    
    audioWin = new Audio(SOUNDS.win);
    audioWin.volume = 0.4;
  });

  function playSfx(type: 'spin' | 'stop' | 'win') {
    if (isMuted) return;
    
    if (type === 'spin') {
      audioSpin.currentTime = 0;
      audioSpin.play().catch(() => {});
    } else if (type === 'stop') {
      const stopClone = audioStop.cloneNode() as HTMLAudioElement;
      stopClone.volume = 0.3;
      stopClone.play().catch(() => {});
    } else if (type === 'win') {
      audioWin.currentTime = 0;
      audioWin.play().catch(() => {});
    }
  }

  function stopSfx(type: 'spin') {
    if (type === 'spin' && audioSpin) {
      audioSpin.pause();
    }
  }

  function getRandomSymbols() {
    return Array(ROWS).fill(0).map(() => SYMBOLS[Math.floor(Math.random() * SYMBOLS.length)]);
  }

  async function spin() {
    if (spinning || balance < bet) return;

    spinning = true;
    balance -= bet;
    lastWin = 0;
    playSfx('spin');

    // Simulate spin delay with staggered reel stops
    for (let i = 0; i < REEL_COUNT; i++) {
      await new Promise(resolve => setTimeout(resolve, 600 + i * 400));
      reels[i] = getRandomSymbols();
      playSfx('stop');
    }

    stopSfx('spin');
    checkWin();
    spinning = false;
  }

  function checkWin() {
    let winAmount = 0;
    
    // Check horizontal lines
    for (let i = 0; i < ROWS; i++) {
      const row = reels.map(reel => reel[i]);
      if (row.every(symbol => symbol.id === row[0].id)) {
        winAmount += row[0].value * (bet / 10);
      }
    }

    // Check diagonals
    if (reels[0][0].id === reels[1][1].id && reels[1][1].id === reels[2][2].id) {
      winAmount += reels[1][1].value * (bet / 10);
    }
    if (reels[0][2].id === reels[1][1].id && reels[1][1].id === reels[2][0].id) {
      winAmount += reels[1][1].value * (bet / 10);
    }

    if (winAmount > 0) {
      balance += winAmount;
      lastWin = winAmount;
      playSfx('win');
      triggerWinParticles();
      history = [{
        result: reels.map(r => r[1].icon).join(' '),
        win: winAmount,
        time: new Date().toLocaleTimeString()
      }, ...history].slice(0, 10);
    }
  }

  function triggerWinParticles() {
    const newParticles = Array(20).fill(0).map((_, i) => ({
      id: Date.now() + i,
      x: Math.random() * 100,
      y: Math.random() * 100
    }));
    winParticles = newParticles;
    setTimeout(() => winParticles = [], 2000);
  }

  function adjustBet(direction: number) {
    if (spinning) return;
    const newIndex = betMultiplierIndex + direction;
    if (newIndex >= 0 && newIndex < BET_MULTIPLIERS.length) {
      const nextBet = BASE_BET * BET_MULTIPLIERS[newIndex];
      if (nextBet <= balance || direction < 0) {
        betMultiplierIndex = newIndex;
      }
    }
  }

</script>

<main class="min-h-screen bg-neutral-950 text-white font-sans selection:bg-purple-500/30 overflow-x-hidden">
  <!-- Win Particles -->
  {#each winParticles as particle (particle.id)}
    <div 
      class="fixed pointer-events-none z-[100] text-yellow-400"
      style="left: {particle.x}%; top: {particle.y}%;"
      in:scale={{ duration: 500, easing: elasticOut }}
      out:fade={{ duration: 1000 }}
    >
      <Star class="w-6 h-6 fill-current animate-spin" />
    </div>
  {/each}

  <!-- Header -->
  <header class="p-3 md:p-6 flex justify-between items-center border-b border-white/10 bg-black/40 backdrop-blur-xl sticky top-0 z-50">
    <div class="flex items-center gap-2 md:gap-3">
      <div class="w-8 h-8 md:w-10 md:h-10 bg-gradient-to-br from-purple-600 to-blue-600 rounded-lg md:rounded-xl flex items-center justify-center shadow-lg shadow-purple-500/20">
        <Trophy class="w-5 h-5 md:w-6 md:h-6 text-white" />
      </div>
      <h1 class="text-lg md:text-2xl font-bold tracking-tight bg-clip-text text-transparent bg-gradient-to-r from-white to-white/60">
        Svelte Slot
      </h1>
    </div>

    <div class="flex items-center gap-2 md:gap-4">
      <button 
        onclick={() => isMuted = !isMuted}
        class="p-1.5 md:p-2 rounded-full hover:bg-white/5 transition-colors text-white/60 hover:text-white"
        aria-label={isMuted ? "Unmute" : "Mute"}
      >
        {#if isMuted}
          <VolumeX class="w-5 h-5 md:w-6 md:h-6" />
        {:else}
          <Volume2 class="w-5 h-5 md:w-6 md:h-6" />
        {/if}
      </button>

      <div class="flex flex-col items-end min-w-[80px] md:min-w-[120px]">
        <span class="text-[8px] md:text-[10px] uppercase tracking-widest text-white/40 font-bold">Balance</span>
        <div class="flex items-center gap-1 md:gap-2">
          <Coins class="w-3 h-3 md:w-4 md:h-4 text-yellow-400" />
          <span class="text-sm md:text-xl font-mono font-bold text-yellow-400">
            ${Math.floor($displayedBalance).toLocaleString()}
          </span>
        </div>
        {#if balance === 0 && timeUntilRecovery}
          <span class="text-[7px] md:text-[9px] text-red-400 font-black animate-pulse uppercase tracking-tighter">
            {timeUntilRecovery}
          </span>
        {/if}
      </div>
      <button 
        onclick={() => showHistory = !showHistory}
        class="p-1.5 md:p-2 rounded-full hover:bg-white/5 transition-colors text-white/60 hover:text-white"
      >
        <History class="w-5 h-5 md:w-6 md:h-6" />
      </button>
    </div>
  </header>

  <div class="max-w-4xl mx-auto p-3 md:p-8 flex flex-col gap-4 md:gap-8">
    <!-- Slot Machine -->
    <div class="relative group">
      <div class="absolute -inset-1 bg-gradient-to-r from-purple-600 to-blue-600 rounded-[2rem] md:rounded-[2.5rem] blur opacity-20 group-hover:opacity-30 transition duration-1000"></div>
      
      <div class="relative bg-neutral-900 border border-white/10 rounded-[1.5rem] md:rounded-[2rem] overflow-hidden shadow-2xl">
        <!-- Reel Area -->
        <div class="grid grid-cols-3 gap-1.5 md:gap-4 p-3 md:p-8 bg-black/40">
          {#each reels as reel, i}
            <div class="relative h-[220px] sm:h-[280px] md:h-[380px] bg-neutral-800/50 rounded-xl md:rounded-2xl border border-white/5 overflow-hidden">
              <div 
                class="absolute inset-0 flex flex-col justify-around items-center py-2 md:py-4 transition-transform duration-500"
                class:animate-reel-spin={spinning}
                style="animation-delay: {i * 150}ms"
              >
                {#each reel as symbol}
                  <div 
                    class="text-3xl sm:text-4xl md:text-6xl transition-all duration-500 transform"
                    class:blur-md={spinning}
                    class:scale-90={spinning}
                    class:opacity-50={spinning}
                  >
                    {symbol.icon}
                  </div>
                {/each}
              </div>
              <!-- Glass Effect Overlay -->
              <div class="absolute inset-0 pointer-events-none bg-gradient-to-b from-black/60 via-transparent to-black/60"></div>
              <div class="absolute inset-0 pointer-events-none border-x border-white/5"></div>
            </div>
          {/each}
        </div>

        <!-- Win Display -->
        <div class="h-12 md:h-20 bg-neutral-900/80 border-t border-white/10 flex items-center justify-center overflow-hidden">
          {#if lastWin > 0}
            <div 
              class="flex items-center gap-2 md:gap-4"
              in:fly={{ y: 50, duration: 800, easing: elasticOut }}
            >
              <Trophy class="w-5 h-5 md:w-8 md:h-8 text-yellow-400 animate-bounce" />
              <span class="text-xl md:text-4xl font-black text-yellow-400 tracking-tighter">
                WIN! ${lastWin.toLocaleString()}
              </span>
              <Trophy class="w-5 h-5 md:w-8 md:h-8 text-yellow-400 animate-bounce" />
            </div>
          {:else if spinning}
            <div class="flex items-center gap-2" in:fade>
              <RotateCw class="w-3 h-3 md:w-4 md:h-4 text-purple-500 animate-spin" />
              <span class="text-white/40 font-bold tracking-[0.2em] uppercase text-[8px] md:text-[10px]">Spinning</span>
            </div>
          {:else}
            <span class="text-white/20 font-medium tracking-widest uppercase text-[10px] md:text-xs" in:fade>Good Luck!</span>
          {/if}
        </div>
      </div>
    </div>

    <!-- Controls -->
    <div class="grid grid-cols-1 md:grid-cols-3 gap-3 md:gap-6 items-center">
      <!-- Bet Controls -->
      <div class="bg-white/5 p-3 md:p-6 rounded-2xl md:rounded-3xl border border-white/10 flex flex-col gap-1 md:gap-4 order-2 md:order-1">
        <div class="flex justify-between items-center">
          <span class="text-[8px] md:text-[10px] uppercase tracking-widest text-white/40 font-bold">Bet Amount</span>
          <span class="text-[8px] md:text-[10px] uppercase tracking-widest text-purple-400 font-bold">{BET_MULTIPLIERS[betMultiplierIndex]}x Multiplier</span>
        </div>
        <div class="flex items-center justify-between">
          <button 
            onclick={() => adjustBet(-1) }
            disabled={spinning || betMultiplierIndex <= 0}
            class="w-8 h-8 md:w-12 md:h-12 rounded-lg md:rounded-2xl bg-white/5 hover:bg-white/10 disabled:opacity-20 transition-all flex items-center justify-center font-bold text-lg active:scale-90"
          >
            -
          </button>
          <span class="text-xl md:text-4xl font-mono font-bold tracking-tighter">${bet.toLocaleString()}</span>
          <button 
            onclick={() => adjustBet(1)}
            disabled={spinning || betMultiplierIndex >= BET_MULTIPLIERS.length - 1 || (BASE_BET * BET_MULTIPLIERS[betMultiplierIndex + 1]) > balance}
            class="w-8 h-8 md:w-12 md:h-12 rounded-lg md:rounded-2xl bg-white/5 hover:bg-white/10 disabled:opacity-20 transition-all flex items-center justify-center font-bold text-lg active:scale-90"
          >
            +
          </button>
        </div>
      </div>

      <!-- Spin Button -->
      <div class="flex justify-center order-1 md:order-2">
        <button 
          onclick={spin}
          disabled={spinning || balance < bet}
          class="group relative w-24 h-24 md:w-36 md:h-36 rounded-full bg-gradient-to-br from-purple-600 to-blue-600 p-1 shadow-xl md:shadow-2xl shadow-purple-500/30 hover:shadow-purple-500/50 transition-all active:scale-90 disabled:opacity-50 disabled:grayscale"
        >
          <div class="w-full h-full rounded-full bg-neutral-950 flex flex-col items-center justify-center group-hover:bg-transparent transition-colors">
            <RotateCw class="w-8 h-8 md:w-14 md:h-14 text-white {spinning ? 'animate-spin' : 'group-hover:rotate-180 transition-transform duration-700'}" />
            <span class="text-[7px] md:text-[10px] font-black uppercase tracking-widest mt-1 md:mt-2 text-white/60 group-hover:text-white">Spin</span>
          </div>
        </button>
      </div>

      <!-- Quick Stats -->
      <div class="bg-white/5 p-3 md:p-6 rounded-2xl md:rounded-3xl border border-white/10 flex flex-col gap-1 md:gap-4 order-3">
        <span class="text-[8px] md:text-[10px] uppercase tracking-widest text-white/40 font-bold">Multipliers</span>
        <div class="grid grid-cols-4 md:grid-cols-2 gap-x-2 md:gap-x-4 gap-y-1 text-[8px] md:text-[10px] text-white/60">
          <div class="flex justify-between md:block"><span>7️⃣7️⃣7️⃣</span> <span class="text-red-500 font-bold md:ml-1">50x</span></div>
          <div class="flex justify-between md:block"><span>💎💎💎</span> <span class="text-blue-400 font-bold md:ml-1">25x</span></div>
          <div class="flex justify-between md:block"><span>🔔🔔🔔</span> <span class="text-yellow-500 font-bold md:ml-1">10x</span></div>
          <div class="flex justify-between md:block"><span>🍇🍇🍇</span> <span class="text-purple-500 font-bold md:ml-1">5x</span></div>
        </div>
      </div>
    </div>
  </div>

  <!-- History Sidebar -->
  {#if showHistory}
    <div 
      class="fixed inset-0 bg-black/80 backdrop-blur-md z-[60] flex justify-end"
      onclick={() => showHistory = false}
      onkeydown={(e) => e.key === 'Escape' && (showHistory = false)}
      role="button"
      tabindex="0"
      aria-label="Close history"
      transition:fade={{ duration: 300 }}
    >
      <div 
        class="w-full sm:max-w-md bg-neutral-900 h-full p-6 md:p-8 shadow-2xl border-l border-white/10 flex flex-col"
        onclick={e => e.stopPropagation()}
        onkeydown={e => e.stopPropagation()}
        role="presentation"
        transition:fly={{ x: 400, duration: 500, easing: cubicOut }}
      >
        <div class="flex justify-between items-center mb-6 md:mb-10">
          <div class="flex items-center gap-3">
            <History class="w-5 h-5 md:w-6 md:h-6 text-purple-500" />
            <h2 class="text-xl md:text-2xl font-bold tracking-tight">Recent Spins</h2>
          </div>
          <button 
            onclick={() => showHistory = false} 
            class="w-8 h-8 md:w-10 md:h-10 rounded-full bg-white/5 flex items-center justify-center hover:bg-white/10 transition-colors"
          >
            ✕
          </button>
        </div>

        <div class="flex-1 flex flex-col gap-3 md:gap-4 overflow-y-auto pr-2 custom-scrollbar">
          {#each history as item, i (item.time + i)}
            <div 
              class="bg-white/5 p-4 md:p-5 rounded-2xl md:rounded-3xl border border-white/5 flex justify-between items-center group hover:bg-white/10 transition-all"
              in:fly={{ y: 20, delay: i * 50 }}
            >
              <div class="flex flex-col gap-1">
                <span class="text-xl md:text-2xl tracking-widest">{item.result}</span>
                <span class="text-[8px] md:text-[10px] text-white/30 uppercase font-black tracking-widest">{item.time}</span>
              </div>
              <div class="flex flex-col items-end">
                <span class="text-green-400 font-black font-mono text-base md:text-lg">+${item.win}</span>
                <span class="text-[7px] md:text-[8px] text-white/20 uppercase font-bold">Payout</span>
              </div>
            </div>
          {:else}
            <div class="flex-1 flex flex-col items-center justify-center text-white/10">
              <History class="w-12 h-12 md:w-16 md:h-16 mx-auto mb-4 md:mb-6 opacity-10" />
              <p class="font-bold uppercase tracking-widest text-xs md:text-sm">No history found</p>
            </div>
          {/each}
        </div>
      </div>
    </div>
  {/if}
</main>

<style>
  :global(body) {
    background-color: #0a0a0a;
    overflow-x: hidden;
  }

  @keyframes reel-spin {
    0% { transform: translateY(0); }
    100% { transform: translateY(-20px); }
  }

  .animate-reel-spin {
    animation: reel-spin 0.1s linear infinite;
  }

  .custom-scrollbar::-webkit-scrollbar {
    width: 4px;
  }
  .custom-scrollbar::-webkit-scrollbar-track {
    background: transparent;
  }
  .custom-scrollbar::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 10px;
  }
</style>
