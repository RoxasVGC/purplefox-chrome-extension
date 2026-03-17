<template>
  <div class="w-max bg-white text-md px-2 py-3 min-w-[200px]">
    <template v-if="tab === 'main'">
      <p class="text-center" v-if="currentSoftware">
        Currently detecting
        <span class="font-bold text-purple-500">{{ currentSoftware }}</span>
      </p>
      <p class="text-center" v-else>No known software detected</p>

      <div v-if="['rk9', 'purplefox'].includes(currentSoftware || '')">
        <div class="flex justify-center mt-2 px-3">
          <textarea
            id="manualInput"
            class="w-full border p-1 text-xs text-black"
            style="height: 60px; color: black;"
            placeholder="Paste HTML..."
          ></textarea>
        </div>
        <div class="flex justify-center mt-2 mb-4">
          <button
            class="button"
            style="background-color: #10b981; color: white;"
            @click="processManualData"
            :disabled="isLoading"
          >
            Elaborate HTML
          </button>
        </div>
      </div>

      <div v-if="['carde', 'rk9'].includes(currentSoftware || '')" class="flex justify-center mt-2">
        <button
          class="button"
          @click="extractPlayers"
          :disabled="isLoading"
        >
          Extract players
        </button>
      </div>
<div v-if="canExtractHeroes" class="flex justify-center mt-2">
        <button class="button" @click="extractHeroes" :disabled="isLoading">
          Extract heroes
        </button>
      </div>

      <div v-if="canExtractResults" class="flex justify-center mt-2">
        <button class="button" @click="extractResults" :disabled="isLoading">
          Extract results
        </button>
      </div>

      <div v-if="canExtractStandings" class="flex justify-center mt-2">
        <button class="button" @click="extractStanding" :disabled="isLoading">
          Extract standings
        </button>
      </div>


      <p v-else-if="!['rk9', 'purplefox', 'kgcn'].includes(currentSoftware || '')">No action possible on this page</p>

      <div v-if="isLoading" class="loader"></div>
      <p v-if="message" class="mt-2">{{ message }}</p>
      <button
        class="block mt-3 ml-auto text-xs text-purple-500 underline hover:text-purple-700"
        @click="tab = 'settings'"
      >
        Settings
      </button>
    </template>
    <template v-if="tab === 'settings'">
      Currently {{ tournamentsCount }} tournaments are in memory.
      <button
        v-if="tournamentsCount !== 0"
        class="mt-2 button"
        @click="clearMemory"
      >
        Clear
      </button>
      <div>
        <button
          class="block mt-3 ml-auto text-xs text-purple-500 underline hover:text-purple-700"
          @click="tab = 'main'"
        >
          Back
        </button>
      </div>
    </template>
  </div>
</template>

<script lang="ts">
import "./style.css";
import { defineComponent, ref } from "vue";
import { useTournaments } from "@/use/useTournaments";
import { usePath } from "@/use/usePath";

export default defineComponent({
  name: "App",
  setup() {
    const { tournamentsCount, clearMemory } = useTournaments();
    const {
      currentSoftware,
      canExtractResults,
      canExtractHeroes,
      canExtractStandings,
    } = usePath();

    const token = ref("");
    chrome.storage.local.get("token", function ({ token: value }) {
      token.value = value;
      console.log("Loaded token from storage", value);
    });

    return {
      currentSoftware,
      tournamentsCount,
      clearMemory,
      canExtractResults,
      canExtractHeroes,
      canExtractStandings,
      tab: ref("main"),
      message: ref(""),
      isLoading: ref(false),
      token,
    };
  },
  methods: {
    processManualData() {
      this.isLoading = true;
      const manualInput = document.getElementById("manualInput") as HTMLTextAreaElement;
      const rawHtml = manualInput ? manualInput.value : "";
      
      if (!rawHtml) {
        this.message = "Paste HTML first!";
        this.isLoading = false;
        return;
      }

      try {
        const parser = new DOMParser();
        const doc = parser.parseFromString(rawHtml, "text/html");
        const results: any[] = [];
        let errors = 0;
        
        const matchRows = doc.querySelectorAll("div.match"); 
        
        matchRows.forEach(row => {
          try {
            const tableElement = row.querySelector(".tablenumber") as HTMLElement;
            if (!tableElement) return; 
            const tableNumber = parseInt(tableElement.innerText.trim(), 10);
            
            const p1Element = row.querySelector(".player1 .name") as HTMLElement;
            const p2Element = row.querySelector(".player2 .name") as HTMLElement;
            const playerName1 = p1Element ? p1Element.innerText.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() : "";
            const playerName2 = p2Element ? p2Element.innerText.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() : "";
            
            let matchResult = null;
            const p1Div = row.querySelector(".player1") as HTMLElement;
            const p2Div = row.querySelector(".player2") as HTMLElement;
            const p1Text = p1Div ? p1Div.innerText.toLowerCase() : "";
            const p2Text = p2Div ? p2Div.innerText.toLowerCase() : "";
            const rowText = (row as HTMLElement).innerText.toLowerCase();

            if ((p1Div && p1Div.classList.contains("tie")) || rowText.includes("tie submitted")) { 
              matchResult = "DRAW"; 
            } else if ((p1Div && p1Div.classList.contains("winner")) || p1Text.includes("win submitted") || p1Text.includes("result submitted")) { 
              matchResult = "1WIN"; 
            } else if ((p2Div && p2Div.classList.contains("winner")) || p2Text.includes("win submitted") || p2Text.includes("result submitted")) { 
              matchResult = "2WIN"; 
            } else if (rowText.includes("submitted")) { 
              matchResult = "PENDING"; 
            }

            results.push({ tableNumber, playerName1, playerName2, result: matchResult });
          } catch (e) {
            errors++;
          }
        });

        navigator.clipboard.writeText(JSON.stringify(results));
        this.message = errors > 0 ? `${errors} errors. Data copied.` : "Elaboration complete! Copied.";
        if (manualInput) manualInput.value = ""; 

      } catch (err) {
        this.message = "Error elaborating HTML.";
      }
      
      setTimeout(() => { this.message = "" }, 2000);
      this.isLoading = false;
    },
    async extractResults() {
      this.isLoading = true;
      let [tab] = await chrome.tabs.query({
        active: true,
        currentWindow: true,
      });
      if (!tab) return;
      chrome.scripting.executeScript(
        {
          target: { tabId: tab.id as number },
          func: (
            this.currentSoftware === "gem" ? extractResultGem :
            this.currentSoftware === "rk9" ? extractRk9Pairings :
            this.currentSoftware === "kgcn" ? extractResultKgcn :
            extractResultCarde
          ) as any,
          args: [this.token || ""],
        },
        (results: any) => {
          if ((chrome.runtime as any).lastError || !results || !results[0]) {
            this.message = "Script Error";
            this.isLoading = false;
            return;
          }
          const { result } = results[0];
          navigator.clipboard.writeText(JSON.stringify(result.value));
          this.message = result.message;
          if (result.errorCount === 0) {
            setTimeout(() => {
              this.message = "";
            }, 2000);
          }
          this.isLoading = false;
        }
      );
    },
    async extractHeroes() {
      this.isLoading = true;
      let [tab] = await chrome.tabs.query({
        active: true,
        currentWindow: true,
      });
      if (!tab) return;
      chrome.scripting.executeScript(
        {
          target: { tabId: tab.id as number },
          func: exportHeroes as any,
        },
        (results: any) => {
          if ((chrome.runtime as any).lastError || !results || !results[0]) {
            this.message = "Script Error";
            this.isLoading = false;
            return;
          }
          const { result } = results[0];
          this.isLoading = false;
          navigator.clipboard.writeText(JSON.stringify(result));
        }
      );
    },
    async extractStanding() {
      this.isLoading = true;
      let [tab] = await chrome.tabs.query({
        active: true,
        currentWindow: true,
      });
      if (!tab) return;
      chrome.scripting.executeScript(
        {
          target: { tabId: tab.id as number },
          func: (
            this.currentSoftware === "rk9" ? extractRk9Standings : 
            this.currentSoftware === "kgcn" ? extractStandingKgcn :
            extractStandingCarde
          ) as any,
          args: [this.token || ""],
        },
        (results: any) => {
          if ((chrome.runtime as any).lastError || !results || !results[0]) {
            this.message = "Script Error";
            this.isLoading = false;
            return;
          }
          const { result } = results[0];
          navigator.clipboard.writeText(JSON.stringify(result.value));
          this.message = result.message;
          if (result.errorCount === 0) {
            setTimeout(() => {
              this.message = "";
            }, 2000);
          }
          this.isLoading = false;
        }
      );
    },
    async extractPlayers() {
      this.isLoading = true;
      let [tab] = await chrome.tabs.query({
        active: true,
        currentWindow: true,
      });
      if (!tab) return;
      chrome.scripting.executeScript(
        {
          target: { tabId: tab.id as number },
          func: (this.currentSoftware === "rk9" ? extractRk9Players : extractPlayersCarde) as any,
          args: [this.token || ""],
        },
        (results: any) => {
          if ((chrome.runtime as any).lastError || !results || !results[0]) {
            this.message = "Script Error";
            this.isLoading = false;
            return;
          }
          const { result } = results[0];
          navigator.clipboard.writeText(result.value.map((p: any) => `${p.gameId}\t${p.name}`).join("\n"));
          this.message = result.message;
          if (result.errorCount === 0) {
            setTimeout(() => {
              this.message = "";
            }, 2000);
          }
          this.isLoading = false;
        }
      );
    },
  },
});

// --- FUNZIONI ORIGINALI ---

function extractResultGem() {
  const PLAYER_REGEXP = /^(.+) \((.+)\)$/;
  const TRANSLATE: any = {
    "Player 1 Win": "1WIN",
    "Player 2 Win": "2WIN",
    Draw: "DRAW",
  };
  const result: any = {};

  document.querySelectorAll(".match-row").forEach((row) => {
    const cells = row.querySelectorAll(".match-element");
    const [, playerName1 = null, playerGameId1 = null] =
      ((cells[1] as HTMLElement)?.innerText || "").trim().match(PLAYER_REGEXP) || [];
    const [, playerName2 = null, playerGameId2 = null] =
      ((cells[2] as HTMLElement)?.innerText || "").trim().match(PLAYER_REGEXP) || [];

    const tableNumber = parseInt((cells[0] as HTMLElement)?.innerText);
    result[tableNumber] = {
      tableNumber,
      playerName1,
      playerGameId1,
      playerName2,
      playerGameId2,
      result: cells[3]?.querySelector("select")?.value || null,
    };
  });

  let errorCount = 0;
  document.querySelectorAll("#refresh ul li").forEach((line) => {
    if (line.getAttribute("id") === "report-drops") return;
    if (line.getAttribute("id") === "report-undrop") return;
    const text = (line.querySelector("span") as HTMLElement)?.innerText?.trim();
    const [rawTable, rawPlayer1, rawPlayer2] = text?.split("\n") || [];
    const [, rawTableNumber] = rawTable?.match(/Table (\d+)/) || [];
    const tableNumber = parseInt(rawTableNumber);
    if (isNaN(tableNumber)) return;
    const [, playerName1, playerGameId1] =
      rawPlayer1?.match(/^Player 1 (.*) \((.*)\)/) || [];
    const [, playerName2, playerGameId2] =
      rawPlayer2?.match(/^Player 2 (.*) \((.*)\)/) || [];

    const reportedBy1 = rawPlayer1?.match(/reported (.*)$/)?.[1];
    const reportedBy2 = rawPlayer2?.match(/reported (.*)$/)?.[1];

    if (
      reportedBy1 !== reportedBy2 &&
      reportedBy1 !== "None" &&
      reportedBy2 !== "None"
    ) {
      errorCount = errorCount + 1;
      result[tableNumber] = {
        tableNumber,
        playerName1,
        playerName2,
        playerGameId1,
        playerGameId2,
        result: null,
      };
      return;
    }
    const finalResult = reportedBy1 === "None" ? reportedBy2 : reportedBy1;
    result[tableNumber] = {
      tableNumber,
        playerName1,
      playerName2,
      playerGameId1,
      playerGameId2,
      result: finalResult ? TRANSLATE[finalResult] || finalResult : null,
    };
  });

  let message = "Copied to clipboard";
  if (errorCount > 0) {
    message = `${errorCount} errors found. Copied to clipboard`;
  }

  return { value: Object.values(result), message, errorCount };
}

async function extractResultCarde(token: string) {
  const [, eventId, roundId] =
    window.location.pathname.match(/\/events\/(\d+)\/pairings\/round\/(\d+)/) ||
    [];
  const url = `https://api.admin.carde.io/api/v2/organize/tournament-rounds/${roundId}/matches-list/?round_id=${roundId}&avoid_cache=true&page=1&page_size=3000`;
  const response = await fetch(url, {
    method: "GET",
    headers: {
      Authorization: `Token ${token}`,
    },
    credentials: "include",
  });
  if (!response.ok) {
    return {
      value: [],
      message: `Error fetching data: ${response.statusText}`,
      errorCount: 1,
    };
  }
  const { results: matches } = await response.json();

  const result = matches
    .map((match: any) => {
      if (match.table_number < 0) return;
      const player1 = match?.player_match_relationships?.[0];
      const player2 = match?.player_match_relationships?.[1];
      let matchResult;
      if (match.status === "COMPLETE") {
        matchResult =
          match.winning_player_id === player1?.player?.id
            ? "1WIN"
            : match.winning_player_id === player2?.player?.id
            ? "2WIN"
            : "DRAW";
      }
      return {
        tableNumber: match.table_number,
        playerName1: player1?.player
          ? `${player1.player.last_name}, ${player1.player.first_name}`
          : null,
        playerGameId1: player1?.player?.id || null,
        playerName2: player2?.player
          ? `${player2.player.last_name}, ${player2.player.first_name}`
          : null,
        playerGameId2: player2?.player?.id || null,
        result: matchResult,
      };
    })
    .filter((m: any) => m !== undefined);

  let errorCount = 0;
  let message = "Copied to clipboard";
  if (errorCount > 0) {
    message = `${errorCount} errors found. Copied to clipboard`;
  }
  return { value: result, message, errorCount: 0 };
}

async function extractStandingCarde(token: string) {
  const [, eventId, roundId] =
    window.location.pathname.match(
      /\/events\/(\d+)\/standings\/round\/(\d+)/
    ) || [];
  const url = `https://api.admin.carde.io/api/v2/organize/tournament-rounds/${roundId}/standings?avoid_cache=true&page=1&page_size=3000`;
  const response = await fetch(url, {
    method: "GET",
    headers: {
      Authorization: `Token ${token}`,
    },
    credentials: "include",
  });
  if (!response.ok) {
    return {
      value: [],
      message: `Error fetching data: ${response.statusText}`,
      errorCount: 1,
    };
  }
  const { results: players } = await response.json();

  const result = players.map((line: any) => {
    return {
      name: line.user_event_status?.user?.last_first,
      gameId: line.player?.id,
      standing: line.points,
      isDropped: line.user_event_status?.registration_status === "DROPPED",
      rank: line?.rank,
    };
  });

  let errorCount = 0;
  let message = "Copied to clipboard";
  if (errorCount > 0) {
    message = `${errorCount} errors found. Copied to clipboard`;
  }
  return { value: result, message, errorCount: 0 };
}

async function extractPlayersCarde(token: string) {
  const [, eventId] = window.location.pathname.match(/\/events\/(\d+)/) || [];
  const url = `https://api.admin.carde.io/api/v2/organize/events/${eventId}/registrations-slim?avoid_cache=true&page=1&include_deaths=true&page_size=5000`;
  const response = await fetch(url, {
    method: "GET",
    headers: {
      Authorization: `Token ${token}`,
    },
    credentials: "include",
  });
  if (!response.ok) {
    return {
      value: [],
      message: `Error fetching data: ${response.statusText}`,
      errorCount: 1,
    };
  }
  const { results: players } = await response.json();

  const result = players.filter((line: any) => line.registration_status === 'COMPLETE' || line.registration_status === 'ELIMINATED').map((line: any) => {
    return {
      name: line?.user?.last_first,
      gameId: line.user?.id,
    };
  });

  let errorCount = 0;
  let message = "Copied to clipboard";
  if (errorCount > 0) {
    message = `${errorCount} errors found. Copied to clipboard`;
  }
  return { value: result, message, errorCount: 0 };
}

function exportHeroes() {
  const PLAYER_REGEXP = /^\s+(.+?) \((\d+)\)/;
  const HERO_REGEXP = /^\s+(.+)\n/;
  const result: any[] = [];
  document.querySelectorAll("ol li div.row").forEach((player) => {
    const cells = player.querySelectorAll("div");
    const [, playerName = null, playerGameId = null] =
      cells[0].children[0].innerHTML.match(PLAYER_REGEXP) || [];
    const [, hero = null] = cells[1].innerHTML.match(HERO_REGEXP) || [];
    result.push({
      name: playerName,
      gameId: playerGameId,
      hero: hero,
    });
  });
  return { value: result, message: "Heroes Copied", errorCount: 0 };
}


// --- NUOVE FUNZIONI RK9 (POKEMON) ---

function extractRk9Pairings() {
    const results: any[] = [];
    let errors = 0;
    
    const matchRows = document.querySelectorAll("div.match"); 
    
    matchRows.forEach((row) => {
        try {
            const tableElement = row.querySelector(".tablenumber") as HTMLElement;
            if (!tableElement) return; 
            
            const tableNumber = parseInt(tableElement.innerText.trim(), 10);
            
            const p1Element = row.querySelector(".player1 .name") as HTMLElement;
            const p2Element = row.querySelector(".player2 .name") as HTMLElement;
            
            const playerName1 = p1Element ? p1Element.innerText.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() : "";
            const playerName2 = p2Element ? p2Element.innerText.replace(/\n/g, ' ').replace(/\s+/g, ' ').trim() : "";
            
            let matchResult = null;
            const p1Div = row.querySelector(".player1") as HTMLElement;
            const p2Div = row.querySelector(".player2") as HTMLElement;
            
            const p1Text = p1Div ? p1Div.innerText.toLowerCase() : "";
            const p2Text = p2Div ? p2Div.innerText.toLowerCase() : "";
            const rowText = (row as HTMLElement).innerText.toLowerCase();

            if ((p1Div && p1Div.classList.contains("tie")) || rowText.includes("tie submitted")) {
                matchResult = "DRAW"; 
            } else if ((p1Div && p1Div.classList.contains("winner")) || p1Text.includes("win submitted") || p1Text.includes("result submitted")) { 
                matchResult = "1WIN"; 
            } else if ((p2Div && p2Div.classList.contains("winner")) || p2Text.includes("win submitted") || p2Text.includes("result submitted")) { 
                matchResult = "2WIN"; 
            } else if (rowText.includes("submitted")) { 
                matchResult = "PENDING"; 
            }

            results.push({
                tableNumber: tableNumber,
                playerName1: playerName1,
                playerName2: playerName2,
                result: matchResult
            });

        } catch (e) {
            errors++;
        }
    });

    const finalMessage = errors > 0 
        ? `${errors} errors found. Data copied.` 
        : "RK9 Extraction complete. Copied.";

    return { 
        value: results, 
        message: finalMessage, 
        errorCount: errors 
    };
}

function extractRk9Standings() {
    return { value: [], message: "RK9 Standings in progress!", errorCount: 0 };
}

function extractRk9Players() {
    return { value: [], message: "RK9 Players in progress!", errorCount: 0 };
}

async function extractResultKgcn() {
  // 1. Estrapola l'Event ID dall'URL (es: https://shp.cardgame-network.konami.net/mt/home/#/tournament-duel/E25-387113)
  const match = window.location.hash.match(/\/tournament-duel\/([^\/]+)/);
  if (!match) {
    return { value: [], message: "Tournament ID not found in URL", errorCount: 1 };
  }
  
  const eventId = match[1];
  const url = `https://shp.cardgame-network.konami.net/mt/tournament-underway/round/${eventId}/0?upd=off`;

  try {
    const response = await fetch(url, {
      method: "GET",
      // includiamo le credenziali nel caso in cui l'API si basi sui cookie di sessione di Konami
      credentials: "include", 
    });

    if (!response.ok) {
      return { value: [], message: `Error fetching data: ${response.statusText}`, errorCount: 1 };
    }

    const json = await response.json();
    const compeList = json.data?.compeList || [];

    const results = compeList.map((matchData: any) => {
      let matchResult = "PENDING";
      
      // Traduzione dello status
      if (matchData.result === "WIN") {
        matchResult = "1WIN";
      } else if (matchData.result === "LOSE") {
        matchResult = "2WIN";
      } else if (matchData.result === "BOTH_DEFEAT") {
        matchResult = "DRAW";
      }

      return {
        playerName1: matchData.user1 ? matchData.user1.name : "",
        playerName2: matchData.user2 ? matchData.user2.name : "",
        result: matchResult,
        tableNumber: parseInt(matchData.displayTableName, 10) || 0
      };
    });

    return { 
      value: results, 
      message: "KGCN Extraction complete. Copied.", 
      errorCount: 0 
    };

  } catch (error: any) {
    return { value: [], message: `Error: ${error.message}`, errorCount: 1 };
  }
}

async function extractStandingKgcn() {
  // Estrapola l'Event ID dall'URL
  const match = window.location.hash.match(/\/(?:tournament|tournament-duel)\/([^\/]+)/);
  if (!match) {
    return { value: [], message: "Tournament ID not found in URL", errorCount: 1 };
  }
  
  const eventId = match[1];
  // Utilizziamo il link dell'API che mi hai fornito
  const url = `https://shp.cardgame-network.konami.net/mt/tournament/${eventId}/ranking`;

  try {
    const response = await fetch(url, {
      method: "GET",
      credentials: "include", 
    });

    if (!response.ok) {
      return { value: [], message: `Error fetching data: ${response.statusText}`, errorCount: 1 };
    }

    const json = await response.json();
    // A seconda di come risponde Konami, i dati potrebbero essere nell'array principale o in un oggetto .data
    const rankingList = Array.isArray(json) ? json : (json.data || []);

    const results = rankingList.map((player: any) => {
      // Calcolo dei punti: Vittoria = 3, Pareggio = 1
      const points = (player.wins * 3) + (player.draws * 0);
      
      return {
        gameId: player.cossyId, // Lo manteniamo stringa per preservare eventuali zeri iniziali del cossy ID
        isDropped: player.leavingAwayStatus === 9,
        name: player.cossyName,
        rank: player.rankNo,
        standing: points
      };
    });

    return { 
      value: results, 
      message: "KGCN Standings Extraction complete. Copied.", 
      errorCount: 0 
    };

  } catch (error: any) {
    return { value: [], message: `Error: ${error.message}`, errorCount: 1 };
  }
}

</script>

<style scoped>
.button {
  @apply block bg-purple-500 text-white font-bold py-2 px-4 rounded hover:bg-purple-700 active:bg-transparent active:border-purple-500 active:text-purple-500 border border-transparent;
}
.loader {
  animation: rotate 2s infinite;
  height: 50px;
  width: 50px;
}

.loader:before,
.loader:after {
  border-radius: 50%;
  content: "";
  display: block;
  height: 20px;
  width: 20px;
}
.loader:before {
  animation: ball1 2s infinite;
  @apply bg-purple-500;
  box-shadow: 30px 0 0 #ffa317;
  margin-bottom: 10px;
}
.loader:after {
  animation: ball2 2s infinite;
  background-color: "#FFA317";
  box-shadow: 30px 0 0 #814bb8;
}

@keyframes rotate {
  0% { transform: rotate(0deg) scale(0.8); }
  50% { transform: rotate(360deg) scale(1.2); }
  100% { transform: rotate(720deg) scale(0.8); }
}

@keyframes ball1 {
  0% { box-shadow: 30px 0 0 #ffa317; }
  50% { box-shadow: 0 0 0 #ffa317; margin-bottom: 0; transform: translate(15px, 15px); }
  100% { box-shadow: 30px 0 0 #ffa317; margin-bottom: 10px; }
}

@keyframes ball2 {
  0% { box-shadow: 30px 0 0 #814bb8; }
  50% { box-shadow: 0 0 0 #814bb8; margin-top: -20px; transform: translate(15px, 15px); }
  100% { box-shadow: 30px 0 0 #814bb8; margin-top: 0; }
}
</style>