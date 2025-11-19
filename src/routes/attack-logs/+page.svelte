<script lang="ts">
  import { Tooltip } from "bits-ui";
  import { flyAndScale } from "$lib/utils";
  import { invokeApiRequest } from "$lib/utils/invokeApiRequest";
  import { Method } from "$lib/enums/Method";
  import { onMount, tick, onDestroy } from "svelte";
  import { fly } from "svelte/transition";
  import { user } from '$lib/stores/userStore';
  import {
    ChartBar,
    X,
    ArrowsClockwise,
  } from "phosphor-svelte";
  import { formatDistanceToNow, isDate } from 'date-fns';

  interface AttackLogEntry {
    id: number;
    attacker_userid: number;
    attacker_username: string;
    attacker_pic_square?: string;
    defender_userid: number;
    defender_username: string;
    defender_pic_square?: string;
    type: 'inferno' | 'outpost' | 'main yard' | string;
    x?: number;
    y?: number;
    attackreport: string;
    attacktime: Date | string;
    loot?: any;
  }

  interface AttackLogResponse {
    attackLogs: AttackLogEntry[];
  }

  /**
   * Represents the currently selected filter for attack logs.
   * Corresponds to the API query parameter `filter`.
   */
  let attackLogFilter: 'both' | 'myattacks' | 'peopleattackingme' = 'both';
  let attackLogs: AttackLogEntry[] = [];
  let isFetching = true;
  let lastRefreshTime: Date | null = null; // Track the last refresh time

  let showModal = false;
  let selectedAttackReport: AttackLogEntry | null = null;

  const filterOptions = [
    { value: 'both', label: 'Both' },
    { value: 'myattacks', label: 'My Attacks' },
    { value: 'peopleattackingme', label: 'People Attacking Me' },
  ];

  // --- Functions ---
  /**
   * Converts the attack log entry's attacktime to a human-readable "time ago" string.
   * @param time The attacktime value.
   * @returns A string representing the time elapsed since the attack.
   */
  const formatTimeAgo = (time: Date | string): string => {
    let date: Date;

    if (time instanceof Date) {
      date = time;
    } else {
      date = new Date(time);
    }

    if (!isDate(date) || isNaN(date.getTime())) {
      return 'Unknown Time';
    }

    return formatDistanceToNow(date, { addSuffix: true });
  };


  /**
   * Constructs the log description based on the attacker/defender relative to the authenticated user.
   * @param log The AttackLogEntry.
   * @returns The descriptive string.
   */
  const getLogDescription = (log: AttackLogEntry): string => {
    // NOTE: Assuming $user is correctly populated with the current user's data, including userid.
    const currentUserId = $user.userid;

    if (log.attacker_userid === currentUserId) {
      return `You attacked ${log.defender_username}`;
    } else if (log.defender_userid === currentUserId) {
      return `${log.attacker_username} attacked you`;
    }
    return 'Attack Log Entry'; 
  };

  /**
   * Constructs the location string based on the attack type and coordinates.
   * @param log The AttackLogEntry.
   * @returns The location string or a dash.
   */
  const getLogLocation = (log: AttackLogEntry): string => {
    if (log.type && log.type.toLowerCase() === 'inferno') {
      return '-';
    }
    if (log.x !== undefined && log.y !== undefined) {
      return `${log.x}x${log.y}`;
    }
    return '-';
  };

  /**
   * Fetches attack logs from the API based on the current filter.
   */
  const fetchAttackLogs = async () => {
    isFetching = true;
    
    // Construct the API URL with the current filter
    const apiUrl = `/attacklogs?filter=${attackLogFilter}`;

    try {
      const { data } = await invokeApiRequest<AttackLogResponse>(
        apiUrl,
        null,
        Method.GET,
        $user.token
      );
      
      attackLogs = data.attackLogs.map(log => ({
        ...log,
        attacktime: new Date(log.attacktime),
        type: log.type ? log.type.toLowerCase() : 'outpost' 
      }));
      
      lastRefreshTime = new Date(); 

    } catch (err) {
      console.error("Error fetching attack logs:", err);
    } finally {
      isFetching = false;
    }
  };

  /**
   * Handles button click to change the filter and fetch new logs.
   * @param filter The new filter value.
   */
  const handleFilterChange = async (filter: 'both' | 'myattacks' | 'peopleattackingme') => {
    if (attackLogFilter !== filter) {
      attackLogFilter = filter;
      await fetchAttackLogs();
      await tick(); 
      document.getElementById('attack-logs-container')?.scrollIntoView({ behavior: 'smooth' });
    }
  };
  
  /**
   * Opens the modal to display the full attack report.
   * @param log The AttackLogEntry to display.
   */
  const openReportModal = (log: AttackLogEntry) => {
    selectedAttackReport = log;
    showModal = true;
  };
  
  const closeModal = () => {
    showModal = false;
    selectedAttackReport = null;
  };
  
  let refreshInterval: number;
  const REFRESH_INTERVAL_MS = 30 * 60 * 1000;

  onMount(async () => {
    await fetchAttackLogs();

    refreshInterval = window.setInterval(fetchAttackLogs, REFRESH_INTERVAL_MS);
  });

  onDestroy(() => {
    if (refreshInterval) {
      clearInterval(refreshInterval);
    }
  });

</script>

<svelte:head>
  <title>Attack Logs</title>
</svelte:head>

<div class="mb-16 flex justify-center items-start lg:py-16 lg:mt-[6%] lg:py-0">
  <div class="w-full lg:w-4/5 mx-4 lg:ml-[12%] lg:mr-0">
    <div 
      class="flex flex-col items-center text-muted-foreground"
      in:fly={{ y: 30, duration: 600, delay: 100 }}
    >
      <h1
        class="text-white font-title leading-snug pt-12 text-5xl lg:text-7xl lg:pt-0"
      >
        Attack Logs
      </h1>
    </div>

    <div 
      class="mt-16 flex flex-col gap-4 lg:gap-4"
      in:fly={{ y: 30, duration: 600, delay: 200 }}
    >
      <div class="flex flex-wrap items-center gap-3">
        {#each filterOptions as option}
          <button
            class="px-6 py-2 rounded-md font-bold transition-colors duration-200 {attackLogFilter === option.value
              ? 'bg-primary text-background'
              : 'bg-white/10 text-white hover:bg-white/20'
            }"
            on:click={() => handleFilterChange(option.value as 'both' | 'myattacks' | 'peopleattackingme')}
            disabled={isFetching}
          >
            {option.label}
          </button>
        {/each}

        <button
          class="ml-auto w-10 h-10 rounded-full bg-primary flex items-center justify-center transition-colors duration-200"
          class:bg-primary-dark={isFetching}
          on:click={fetchAttackLogs}
          disabled={isFetching}
          aria-label="Refresh Attack Logs"
        >
          <ArrowsClockwise 
            size={20} 
            weight="bold" 
            class={`text-background ${isFetching ? 'animate-spin' : ''}`}          />
        </button>

        <div class="hidden lg:block">
          <Tooltip.Root openDelay={0}>
            <Tooltip.Trigger>
              <div
                class="w-10 h-10 rounded-full bg-white/10 flex items-center justify-center hover:bg-white/20 transition-all"
              >
                <ChartBar size={20} weight="bold" class="text-primary" />
              </div>
            </Tooltip.Trigger>
            <Tooltip.Content side="top" sideOffset={5}>
              <div class="rounded-[2px]" />
              <div
                class="rounded-input bg-white/10 rounded-md p-3 text-sm font-medium outline-none text-white"
              >
                Attack logs show the latest 50 entries
              </div>
            </Tooltip.Content>
          </Tooltip.Root>
        </div>
      </div>
      
      <div class="text-sm text-gray-400 mt-2">
        {#if isFetching}
          <span class="text-primary font-bold">Loading data...</span>
        {:else if lastRefreshTime}
          <span>Last refresh: {formatTimeAgo(lastRefreshTime)}</span>
        {/if}
      </div>


      <div 
        id="attack-logs-container" 
        class="bg-gray-800 p-5 relative overflow-x-auto rounded-lg shadow-xl"
      >
        <h2 class="font-title text-center text-white text-2xl mb-4">
            Recent Attacks
        </h2>
        
        {#if isFetching && attackLogs.length === 0}
          <div class="flex justify-center items-center py-10">
            <p class="text-primary text-lg">Loading Attack Logs...</p>
          </div>
        {:else if attackLogs.length === 0}
          <div class="flex justify-center items-center py-10">
            <p class="text-muted-foreground">No attack logs found for this filter.</p>
          </div>
        {:else}
          <table class="min-w-full divide-y divide-gray-700">
            <thead>
              <tr>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider"></th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider">Description</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider">Type</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider">Location</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider">Time</th>
                <th class="px-6 py-3 text-left text-xs font-medium text-gray-400 uppercase tracking-wider">Report</th>
              </tr>
            </thead>
            <tbody class="divide-y divide-gray-700">
              {#each attackLogs as log (log.id)}
                <tr transition:fly={{ x: -20, duration: 400 }}>
                  <td class="px-6 py-4 whitespace-nowrap">
                    <img
                      src={log.attacker_pic_square || `https://api.dicebear.com/9.x/bottts-neutral/svg?seed=${log.attacker_username}&size=32`}
                      alt={`${log.attacker_username}'s avatar`}
                      class="w-8 h-8 rounded-sm bg-gray-700 object-cover"
                    />
                  </td>
                  <td class="px-6 py-4 whitespace-nowrap text-sm text-white">
                    {getLogDescription(log)}
                  </td>
                  <td class="px-6 py-4 whitespace-nowrap text-sm text-primary capitalize">
                    {log.type}
                  </td>
                  <td class="px-6 py-4 whitespace-nowrap text-sm text-white">
                    {getLogLocation(log)}
                  </td>
                  <td class="px-6 py-4 whitespace-nowrap text-sm text-white">
                    {formatTimeAgo(log.attacktime)}
                  </td>
                  <td class="px-6 py-4 whitespace-nowrap text-sm font-medium">
                    <button 
                      on:click={() => openReportModal(log)}
                      class="text-primary hover:text-primary-dark transition-colors duration-150"
                    >
                      View Details
                    </button>
                  </td>
                </tr>
              {/each}
            </tbody>
          </table>
        {/if}
      </div>
    </div>
  </div>
</div>

{#if showModal && selectedAttackReport}
  <div 
    class="fixed inset-0 z-50 flex items-center justify-center bg-black/70 p-4"
    transition:flyAndScale
    on:click|self={closeModal}
  >
    <div 
      class="bg-gray-100 rounded-lg shadow-2xl w-full max-w-2xl max-h-[90vh] overflow-hidden flex flex-col"
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
    >
      <div class="flex justify-between items-center p-4 border-b border-gray-300 bg-white">
        <h3 id="modal-title" class="text-xl font-bold text-gray-800">
          Attack Report: {getLogDescription(selectedAttackReport)}
        </h3>
        <button on:click={closeModal} class="text-gray-500 hover:text-gray-700">
          <X size={24} weight="bold" />
        </button>
      </div>

      <div class="p-6 overflow-y-auto flex-grow bg-cream-light">
        <div class="prose max-w-none text-gray-800">
          {@html selectedAttackReport.attackreport}
        </div>
        {#if selectedAttackReport.loot}
            <div class="mt-4 pt-4 border-t border-gray-300">
                <h4 class="font-bold text-gray-800">Loot Details (JSON):</h4>
                <pre class="bg-white p-3 rounded text-sm overflow-x-auto">{JSON.stringify(selectedAttackReport.loot, null, 2)}</pre>
            </div>
        {/if}
      </div>
    </div>
  </div>
{/if}

<style>
  .bg-cream-light {
    background-color: #f5f5dc;
  }

  th:nth-child(1) { width: 5%; }
  th:nth-child(2) { width: 30%; }
  th:nth-child(3) { width: 10%; }
  th:nth-child(4) { width: 15%; }
  th:nth-child(5) { width: 20%; }
  th:nth-child(6) { width: 15%; }
</style>