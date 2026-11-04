<script lang="ts">
    import { onMount } from "svelte";
    import { CurrentFilter, Filters, SortBy, DisplayStyle, SearchQuery } from "@components/store";
    import { scopedTermBeforeCaret } from "@utils/search";

    interface FilterItem {
        name: string;
        displayName?: string;
        class?: string;
        icon?: string;
        count?: number;
    }

    let {
        filter = [],
        sortBy = [],
        displayStyle = [],
        filterName = () => "Status",
        suggestions,
    } = $props<{
        filter: FilterItem[];
        sortBy: string[];
        displayStyle: { name: string; icon: string }[];
        filterName?: () => string;
        /** Opt-in autocomplete for `k:`/`n:` scoped search terms. */
        suggestions?: { names?: string[]; keywords?: string[] };
    }>();

    const MAX_SUGGESTIONS = 8;
    let searchInput: HTMLInputElement | undefined = $state();
    // The `prefix:value` token under the caret, if it is one we can complete
    let activeToken: { start: number; end: number; field: "name" | "keywords"; value: string } | null = $state(null);
    let highlighted = $state(0);

    let suggestionList = $derived.by(() => {
        if (!activeToken || !suggestions) return [];
        const pool = (activeToken.field === "name" ? suggestions.names : suggestions.keywords) ?? [];
        const needle = activeToken.value.toLowerCase();
        // One pass: prefix matches first, then substring matches, stopping once there are enough prefix matches
        const starts: string[] = [];
        const contains: string[] = [];
        for (const item of pool) {
            const index = item.toLowerCase().indexOf(needle);
            if (index === 0) starts.push(item);
            else if (index > 0) contains.push(item);
            if (starts.length === MAX_SUGGESTIONS) break;
        }
        return [...starts, ...contains].slice(0, MAX_SUGGESTIONS);
    });
    const comboboxAttrs = $derived(
        suggestions
            ? {
                  role: "combobox",
                  "aria-autocomplete": "list" as const,
                  "aria-expanded": suggestionList.length > 0,
                  "aria-controls": "search-suggestions",
              }
            : {},
    );

    function updateActiveToken() {
        if (!suggestions || !searchInput) return;
        const caret = searchInput.selectionStart ?? searchInput.value.length;
        const term = scopedTermBeforeCaret(searchInput.value.slice(0, caret));
        if (!term || (term.field !== "name" && term.field !== "keywords")) {
            activeToken = null;
            return;
        }
        // Extend over the rest of the token after the caret, so accepting replaces all of it
        const rest = /^\S*/.exec(searchInput.value.slice(caret))![0];
        activeToken = { start: term.start, end: caret + rest.length, field: term.field, value: term.value };
        highlighted = 0;
    }

    function acceptSuggestion(value: string) {
        if (!activeToken || !searchInput) return;
        // Quote only when needed, otherwise a space would split the value into two search terms
        const term = `${activeToken.field === "keywords" ? "k" : "n"}:${value.includes(" ") ? `"${value}"` : value}`;
        const current = searchInput.value;
        const after = current.slice(activeToken.end).replace(/^\s*/, "");
        const next = `${current.slice(0, activeToken.start)}${term} ${after}`;
        searchInput.value = next;
        // Not trimmed: the input mirrors $SearchQuery, so trimming would eat the trailing space and move the caret
        SearchQuery.set(next);
        const caret = activeToken.start + term.length + 1;
        searchInput.setSelectionRange(caret, caret);
        searchInput.focus();
        activeToken = null;
    }

    function handleSearchKeydown(e: KeyboardEvent) {
        if (suggestionList.length === 0) return;
        if (e.key === "ArrowDown") {
            e.preventDefault();
            highlighted = (highlighted + 1) % suggestionList.length;
        } else if (e.key === "ArrowUp") {
            e.preventDefault();
            highlighted = (highlighted - 1 + suggestionList.length) % suggestionList.length;
        } else if (e.key === "Enter" || e.key === "Tab") {
            e.preventDefault();
            acceptSuggestion(suggestionList[highlighted]);
        } else if (e.key === "Escape") {
            activeToken = null;
        }
    }
    // Initialize filters once on mount
    onMount(async () => {
        // Reset stores if no filters provided
        if (filter.length === 0) {
            Filters.set([]);
            CurrentFilter.set([]);
        } else {
            Filters.set(filter);
        }

        // Reset sort if no options provided
        if (sortBy.length === 0) {
            SortBy.set("");
        } else if (!$SortBy) {
            SortBy.set(sortBy[0]);
        }

        // Reset display style if no options provided
        if (displayStyle.length === 0) {
            DisplayStyle.set("grid");
        }

        // Always reset search
        SearchQuery.set("");
    });

    function handleSearch(q: Event) {
        const target = q.target as HTMLInputElement;
        SearchQuery.set(target.value.trim());
        updateActiveToken();
    }

    function handleFilter(fil: string, e: Event) {
        e.preventDefault();
        // remove focus from button
        (e.target as HTMLElement).blur();

        const newFilters = $CurrentFilter.some((f) => f.name === fil)
            ? $CurrentFilter.filter((f) => f.name !== fil)
            : [...$CurrentFilter, { name: fil }];

        CurrentFilter.set(newFilters);
    }

    function handleExclusiveFilter(fil: string, e: Event) {
        e.preventDefault();
        // remove focus from button
        (e.target as HTMLElement).blur();
        CurrentFilter.set([{ name: fil }]);
    }

    function handleSort(sor: string) {
        SortBy.set(sor);
    }

    function handleDisplayStyle(style: string) {
        DisplayStyle.set(style);
    }
</script>

<div class="filter-bar mb-2 px-2">
    <div class="d-flex w-100 justify-content-between justify-content-md-center">
        <div class="position-relative w-25 me-2">
            <input
                type="text"
                class="form-control searchbar"
                bind:this={searchInput}
                value={$SearchQuery}
                oninput={handleSearch}
                onkeydown={handleSearchKeydown}
                onclick={updateActiveToken}
                onblur={() => (activeToken = null)}
                placeholder="Search"
                autocomplete="off"
                {...comboboxAttrs}
            />
            {#if suggestionList.length > 0}
                <ul class="dropdown-menu show w-100 search-suggestions" id="search-suggestions" role="listbox">
                    {#each suggestionList as suggestion, i (suggestion)}
                        <li role="option" aria-selected={i === highlighted}>
                            <button
                                type="button"
                                class="dropdown-item text-truncate"
                                class:active={i === highlighted}
                                onmousedown={(e) => {
                                    // mousedown + preventDefault so the input's blur doesn't close the list first
                                    e.preventDefault();
                                    acceptSuggestion(suggestion);
                                }}
                                onmouseenter={() => (highlighted = i)}>{suggestion}</button
                            >
                        </li>
                    {/each}
                </ul>
            {/if}
        </div>

        {#if filter.length > 0}
            <div class="d-none d-xl-block ms-3 d-flex align-items-center">
                <div class="btn-group ms-1 filter-buttons d-flex" role="group" aria-label="Filter listing">
                    {#each $Filters as fil}
                        <button
                            type="button"
                            data-bs-toggle="tooltip"
                            data-bs-placement="top"
                            data-bs-delay="500"
                            title="Double click to only show items from this category"
                            class={fil.class
                                ? "btn text-nowrap flex-fill btn-outline-" + fil.class
                                : "btn text-nowrap w-100 btn-outline-success"}
                            class:active={$CurrentFilter.some((f) => f.name === fil.name)}
                            onclick={(e) => handleFilter(fil.name, e)}
                            ondblclick={(e) => handleExclusiveFilter(fil.name, e)}
                        >
                            {#if fil.icon}
                                <i class={fil.icon + " me-1"}></i>
                            {/if}
                            {fil.displayName || fil.name}
                            {#if fil.count !== undefined && fil.count >= 0}
                                <span class="badge bg-secondary ms-1">{fil.count}</span>
                            {/if}
                        </button>
                    {/each}
                </div>
            </div>
            <div class="d-xl-none ms-1 ms-md-3 align-items-center">
                <div class="dropdown">
                    <button
                        class="btn btn-outline-success dropdown-toggle text-nowrap"
                        type="button"
                        data-bs-toggle="dropdown"
                        aria-expanded="false"
                    >
                        {filterName()}
                    </button>
                    <ul class="dropdown-menu">
                        {#each $Filters as fil}
                            <li>
                                <div
                                    class="dropdown-item"
                                    title="Filter"
                                    class:active={$CurrentFilter.some((f) => f.name === fil.name)}
                                    onclick={(e) => handleFilter(fil.name, e)}
                                    onkeydown={(e) => e.key === "Enter" && handleFilter(fil.name, e)}
                                    role="button"
                                    tabindex="0"
                                >
                                    {#if fil.icon}
                                        <i class={fil.icon + " fa-fw me-1"}></i>
                                    {/if}
                                    {fil.displayName || fil.name}
                                    {#if fil.count !== undefined && fil.count >= 0}
                                        <span class="badge bg-secondary ms-1">{fil.count}</span>
                                    {/if}
                                </div>
                            </li>
                        {/each}
                    </ul>
                </div>
            </div>
        {/if}
        {#if sortBy.length > 1}
            <div class="ms-1 ms-md-3 align-items-center">
                <div class="dropdown">
                    <button
                        class="btn btn-outline-success dropdown-toggle text-nowrap"
                        type="button"
                        data-bs-toggle="dropdown"
                        aria-expanded="false"
                    >
                        <i class="fa-solid fa-arrow-down-wide-short me-1"></i>
                        <span class="d-none d-md-inline">{$SortBy}</span>
                    </button>
                    <ul class="dropdown-menu">
                        {#each sortBy as sor (sor)}
                            <li>
                                <div
                                    class="dropdown-item"
                                    title="sort"
                                    id={sor.replace(" ", "-")}
                                    class:active={sor === $SortBy}
                                    onclick={() => handleSort(sor)}
                                    onkeydown={() => handleSort(sor)}
                                    role="button"
                                    tabindex="0"
                                >
                                    {sor}
                                </div>
                            </li>
                        {/each}
                    </ul>
                </div>
            </div>
        {/if}
        {#if displayStyle.length > 1}
            <div class="ms-1 ms-md-3 d-flex align-items-center">
                <div class="btn-group ms-1 display-buttons" role="group" aria-label="Display style">
                    {#each displayStyle as dis}
                        <button
                            type="button"
                            class="btn btn-outline-success text-nowrap"
                            onclick={() => handleDisplayStyle(dis.name)}
                            class:active={$DisplayStyle === dis.name}
                            title={dis.name + " view"}
                            data-bs-toggle="tooltip"
                            aria-label={dis.name + " view"}
                            ><i class={dis.icon}></i>
                        </button>
                    {/each}
                </div>
            </div>
        {/if}
    </div>
</div>

<style>
    .search-suggestions {
        max-height: 20rem;
        overflow-y: auto;
    }
</style>
