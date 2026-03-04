<template>
    <div class="relative">
        <v-card
            class="card"
            :class="{ rendered: !storeGraph.loading.value }"
            :elevation="props.flat ? 0 : undefined"
        >
            <v-card-title style="position: relative">
                <v-row
                    density="compact"
                    class="mt-1"
                    style="max-width: calc(100% - 30px)"
                >
                    <v-col md="3" sm="6">
                        <v-autocomplete
                            v-model="form.group"
                            label="Type"
                            no-data-text="No types available"
                            :items="metricGroups"
                            hide-details
                            :disabled="
                                storeGraph.loading.value || !metricGroups.length
                            "
                        >
                        </v-autocomplete>
                    </v-col>
                    <v-col md="4" sm="6">
                        <v-autocomplete
                            v-model="form.metric"
                            label="Metric"
                            no-data-text="No metrics available"
                            :items="metricItems"
                            hide-details
                            :disabled="
                                storeGraph.loading.value || !metricItems.length
                            "
                        >
                            <template v-slot:item="{ props, item }">
                                <v-list-item
                                    v-bind="props"
                                    :title="item.raw.title"
                                    :subtitle="item.raw.description || ''"
                                >
                                    <template #append>
                                        <v-btn
                                            v-if="item.raw.url"
                                            :href="item.raw.url"
                                            target="_blank"
                                            title="Visit documentation for this metric"
                                            icon="$openInNew"
                                            size="x-small"
                                            variant="plain"
                                            class="ml-2 hover-only"
                                        >
                                        </v-btn>
                                    </template>
                                </v-list-item>
                            </template>
                        </v-autocomplete>
                    </v-col>
                    <v-col md="2" sm="6">
                        <v-autocomplete
                            v-model="form.level"
                            label="Level"
                            :disabled="
                                !form.metric ||
                                storeGraph.loading.value ||
                                !metricLevels.length ||
                                props.comparisonMode
                            "
                            :items="metricLevels"
                            hide-details
                        >
                        </v-autocomplete>
                    </v-col>
                    <v-col md="3" sm="6" v-if="hasPhases">
                        <v-menu
                            v-model="phaseMenuOpen"
                            :close-on-content-click="false"
                            location="bottom"
                        >
                            <template v-slot:activator="{ props: menuProps }">
                                <v-text-field
                                    v-bind="menuProps"
                                    label="Filter by Phase"
                                    :model-value="selectedPhasesLabel"
                                    readonly
                                    hide-details
                                    density="compact"
                                    prepend-inner-icon="$filter"
                                    :append-inner-icon="phaseMenuOpen ? '$chevronUp' : '$chevronDown'"
                                    :disabled="storeGraph.loading.value"
                                    clearable
                                    @click:clear="clearPhaseSelection"
                                >
                                </v-text-field>
                            </template>
                            <v-card min-width="300" max-width="400">
                                <v-card-text class="pa-2">
                                    <v-list density="compact" class="pa-0">
                                        <template v-for="group in groupedPhases" :key="group.name">
                                            <v-list-item
                                                :class="{ 'phase-group-expanded': isGroupExpanded(group.name) }"
                                                @click="togglePhaseGroup(group)"
                                            >
                                                <template #prepend>
                                                    <v-checkbox-btn
                                                        :model-value="isGroupFullySelected(group)"
                                                        :indeterminate="isGroupPartiallySelected(group)"
                                                        @click.stop="toggleEntireGroup(group)"
                                                    />
                                                </template>
                                                <v-list-item-title class="d-flex align-center">
                                                    <v-icon
                                                        :color="getLogPhaseColor(group.name)"
                                                        size="small"
                                                        class="mr-2"
                                                        icon="$circle"
                                                    />
                                                    {{ group.name }}
                                                    <v-chip
                                                        size="x-small"
                                                        class="ml-2"
                                                        variant="tonal"
                                                    >
                                                        {{ group.occurrences.length }}x
                                                    </v-chip>
                                                </v-list-item-title>
                                                <template #append>
                                                    <v-icon 
                                                        v-if="group.occurrences.length > 1"
                                                        :icon="isGroupExpanded(group.name) ? '$chevronUp' : '$chevronDown'"
                                                    />
                                                </template>
                                            </v-list-item>
                                            <v-expand-transition>
                                                <div v-show="isGroupExpanded(group.name) && group.occurrences.length > 1">
                                                    <v-list-item
                                                        v-for="occ in group.occurrences"
                                                        :key="`${group.name}-${occ.occurrence}`"
                                                        class="pl-8"
                                                        density="compact"
                                                    >
                                                        <template #prepend>
                                                            <v-checkbox-btn
                                                                :model-value="isPhaseSelected(group.name, occ.occurrence)"
                                                                @click.stop="togglePhaseOccurrence(group.name, occ.occurrence)"
                                                            />
                                                        </template>
                                                        <v-list-item-title>
                                                            #{{ occ.occurrence + 1 }}
                                                            <span class="text-caption text-medium-emphasis ml-2">
                                                                {{ formatDuration(occ.duration_us) }}
                                                            </span>
                                                        </v-list-item-title>
                                                    </v-list-item>
                                                </div>
                                            </v-expand-transition>
                                        </template>
                                    </v-list>
                                </v-card-text>
                            </v-card>
                        </v-menu>
                    </v-col>
                    <v-col md="3" sm="6">
                        <v-autocomplete
                            v-show="form.level !== 'job'"
                            v-model="form.node"
                            label="Node"
                            no-data-text="No nodes available"
                            :items="nodeNames"
                            auto-select-first
                            hide-details
                            :disabled="
                                storeGraph.loading.value ||
                                !nodeNames?.length ||
                                !multiNode
                            "
                        >
                        </v-autocomplete>
                    </v-col>
                </v-row>
                <v-btn
                    class="fullscreen"
                    icon="$fullscreen"
                    variant="text"
                    @click="emit('update:fullscreen', true)"
                    v-show="!props.fullscreen"
                ></v-btn>
            </v-card-title>
            <v-card-text>
                <ReactiveGraph
                    :style="`height: ${props.height}px;`"
                    :graphId="props.graphId"
                    ref="graphRef"
                    :graph="storeGraph.graph"
                    @relayout="emit('relayout', $event)"
                    type="default"
                    :relayoutData="props.relayoutData"
                    @rendered="storeGraph.loading.value = false"
                    :loading="storeGraph.loading.value"
                    :selectedPhases="selectedPhases"
                    v-if="
                        Object.keys(metrics.value || {}).length || !props.noData
                    "
                ></ReactiveGraph>
                <div
                    v-else
                    class="d-flex justify-center no-data"
                    :style="`height: ${props.height}px;`"
                >
                    <div style="margin-top: 160px">no data available</div>
                </div>
                <br>
                <EventGraph 
                    v-if="selectedPhases.find(p => p.name === 'general') ? selectedPhases.length > 1 : selectedPhases.length > 0" 
                    :availablePhases="props.jobPhases" 
                    :selectedPhases="selectedPhases"
                    :relayoutData="props.relayoutData"
                    :jobStartTime="jobStartTime"
                    :interval="graphInterval"
                    :dataCount="graphDataCount"
                    @relayout="emit('relayout', $event)"
                ></EventGraph>

                <!-- graph already displays annotation stating the same issues but graph is never rendered when there are no metics - use this as fallback -->
            </v-card-text>
        </v-card>
    </div>
</template>

<script setup lang="ts">
import { useGraphForm } from "~/components/graphs/useGraphForm";
import { getLogPhaseColor } from "~/components/graphs/useGraphBase";
import type { NodeMap } from "~/repository/modules/nodes";
import type { GraphLevel, Phase } from "~/types/graph";
import type { JobPhase } from "~/repository/modules/jobs";

const { $graphStore } = useNuxtApp();
const graphRef = ref(null);
const { $api } = useNuxtApp();

type RelayoutData = {
    [key: string]: number;
};

// Phase group for the two-level filter UI
interface PhaseGroup {
    name: string;
    expanded: boolean;
    occurrences: Array<{
        occurrence: number;
        started_at: number;
        finished_at: number;
        duration_us: number;
    }>;
}

interface Props {
    graphId: string;
    jobIds?: number[];
    runNr?: number;
    jobPhases?: JobPhase[];
    nodes?: Record<string, NodeMap>;
    relayoutData?: RelayoutData;
    height?: number;
    noData?: boolean;
    defaultGroup?: string;
    defaultMetric?: string;
    defaultLevel?: string;
    flat?: boolean;
    fullscreen?: boolean;
    comparisonMode?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
    jobIds: () => [] as number[],
    runNr: 0,
    jobPhases: () => [] as JobPhase[],
    nodes: () => ({}),
    relayoutData: () => ({}),
    height: 360,
    noData: false,
    defaultGroup: "",
    defaultMetric: "",
    defaultLevel: "",
    flat: false,
    fullscreen: false,
    comparisonMode: false
});

const emit = defineEmits(["update:fullscreen", "relayout"]);

const storeGraph = $graphStore.useStoreGraph(props.graphId, "default");
const metrics = computed(() => storeGraph.metrics.value);
const theme = useCookie("xbat_theme");

// Get job start time from traces (in seconds) for EventGraph
const jobStartTime = computed(() => {
    const firstTrace = Object.values(storeGraph.raw.value)[0]?.traces?.[0];
    if (!firstTrace) return 0;
    const startTimeRaw = firstTrace.start;
    if (!startTimeRaw) return 0;
    
    // Handle both Date objects and ISO strings (from JSON)
    if (startTimeRaw instanceof Date) {
        return startTimeRaw.getTime() / 1000;
    }
    const startTimeString = String(startTimeRaw);
    return new Date(startTimeString.endsWith('Z') ? startTimeString : startTimeString + 'Z').getTime() / 1000;
});

// Get interval from traces for EventGraph (time between data points in seconds)
const graphInterval = computed(() => {
    const firstTrace = Object.values(storeGraph.raw.value)[0]?.traces?.[0];
    return firstTrace?.interval || 5;
});

// Get dataCount from traces for EventGraph (total number of x-axis points)
const graphDataCount = computed(() => {
    const firstTrace = Object.values(storeGraph.raw.value)[0]?.traces?.[0];
    return firstTrace?.values?.length || 0;
});

const { form, metricGroups, metricLevels, metricItems } = useGraphForm(metrics);

// Phase filter state
const phaseMenuOpen = ref(false);
const selectedPhases = ref<Array<{name: string, occurrence: number}>>([]);
const expandedGroups = ref<Set<string>>(new Set());

// Check if phases are available
const hasPhases = computed(() => props.jobPhases && props.jobPhases.length > 0);

// Check if a group is expanded
const isGroupExpanded = (name: string): boolean => {
    return expandedGroups.value.has(name);
};

// Group phases by name for the two-level filter UI
const groupedPhases = computed((): PhaseGroup[] => {
    if (!props.jobPhases?.length) return [];
    
    const groups = new Map<string, PhaseGroup>();
    
    for (const phase of props.jobPhases) {
        if (!groups.has(phase.name)) {
            groups.set(phase.name, {
                name: phase.name,
                expanded: false, // Not used anymore, kept for type compatibility
                occurrences: []
            });
        }
        groups.get(phase.name)!.occurrences.push({
            occurrence: phase.occurrence,
            started_at: phase.started_at,
            finished_at: phase.finished_at,
            duration_us: phase.duration_us
        });
    }
    
    // Sort occurrences within each group
    for (const group of groups.values()) {
        group.occurrences.sort((a, b) => a.occurrence - b.occurrence);
    }
    
    return Array.from(groups.values());
});

// Format duration for display
const formatDuration = (durationUs: number): string => {
    const seconds = durationUs / 1000000;
    if (seconds < 60) return `${seconds.toFixed(1)}s`;
    const minutes = Math.floor(seconds / 60);
    const remainingSeconds = seconds % 60;
    return `${minutes}m ${remainingSeconds.toFixed(0)}s`;
};

// Label for the selected phases input
const selectedPhasesLabel = computed(() => {
    if (selectedPhases.value.length === 0) return '';
    if (selectedPhases.value.length === 1) {
        const sel = selectedPhases.value[0];
        const group = groupedPhases.value.find(g => g.name === sel.name);
        if (group && group.occurrences.length > 1) {
            return `${sel.name} #${sel.occurrence + 1}`;
        }
        return sel.name;
    }
    return `${selectedPhases.value.length} phases selected`;
});

// Check if a specific phase occurrence is selected
const isPhaseSelected = (name: string, occurrence: number): boolean => {
    return selectedPhases.value.some(p => p.name === name && p.occurrence === occurrence);
};

// Check if an entire group is fully selected
const isGroupFullySelected = (group: PhaseGroup): boolean => {
    return group.occurrences.every(occ => isPhaseSelected(group.name, occ.occurrence));
};

// Check if a group is partially selected
const isGroupPartiallySelected = (group: PhaseGroup): boolean => {
    const selectedCount = group.occurrences.filter(occ => 
        isPhaseSelected(group.name, occ.occurrence)
    ).length;
    return selectedCount > 0 && selectedCount < group.occurrences.length;
};

// Toggle expansion of a phase group (only if multiple occurrences)
const togglePhaseGroup = (group: PhaseGroup) => {
    if (group.occurrences.length > 1) {
        const newSet = new Set(expandedGroups.value);
        if (newSet.has(group.name)) {
            newSet.delete(group.name);
        } else {
            newSet.add(group.name);
        }
        expandedGroups.value = newSet;
    } else {
        // If only one occurrence, toggle selection directly
        togglePhaseOccurrence(group.name, group.occurrences[0].occurrence);
    }
};

// Toggle all occurrences in a group
const toggleEntireGroup = (group: PhaseGroup) => {
    const isFullySelected = isGroupFullySelected(group);
    
    if (isFullySelected) {
        // Remove all occurrences of this group
        selectedPhases.value = selectedPhases.value.filter(p => p.name !== group.name);
    } else {
        // Add all missing occurrences
        for (const occ of group.occurrences) {
            if (!isPhaseSelected(group.name, occ.occurrence)) {
                selectedPhases.value.push({ name: group.name, occurrence: occ.occurrence });
            }
        }
    }
    
    fetchPhaseEvents();
};

// Toggle a single phase occurrence
const togglePhaseOccurrence = (name: string, occurrence: number) => {
    const index = selectedPhases.value.findIndex(p => p.name === name && p.occurrence === occurrence);
    
    if (index >= 0) {
        selectedPhases.value.splice(index, 1);
    } else {
        selectedPhases.value.push({ name, occurrence });
    }
    
    fetchPhaseEvents();
};

// Clear all phase selections
const clearPhaseSelection = () => {
    selectedPhases.value = [];
    storeGraph.logData.value = { logs: [], phases: [] };
    storeGraph.updateGraph();
};

// Fetch phase events from API when phases are selected
const fetchPhaseEvents = async () => {
    if (selectedPhases.value.length === 0) {
        storeGraph.logData.value = { logs: [], phases: [] };
        storeGraph.updateGraph();
        return;
    }
    
    const jobId = props.jobIds[0];
    const runNr = props.runNr;
    
    if (!jobId || !runNr) return;
    
    storeGraph.loading.value = true;
    
    try {
        const response = await $api.jobs.getPhaseEvents(jobId, runNr, selectedPhases.value);
        
        if (!response?.phases) {
            storeGraph.logData.value = { logs: [], phases: [] };
            storeGraph.updateGraph();
            return;
        }
        
        // Convert phase events to LogEntry format for graph display
        const logs: Array<{ts: number, hw: string, phase: string | null, msg: string}> = [];
        const phases: Phase[] = [];
        
        for (const phaseData of response.phases) {
            // Add phase info
            phases.push({
                name: phaseData.name,
                occurrence: phaseData.occurrence,
                started_at: phaseData.started_at,
                finished_at: phaseData.finished_at,
                duration_us: phaseData.finished_at - phaseData.started_at
            });
            
            // Convert events to log entries
            for (const event of phaseData.events || []) {
                logs.push({
                    ts: event.ts / 1000000, // Convert microseconds to seconds
                    hw: event.hw,
                    phase: phaseData.name,
                    msg: event.msg
                });
            }
        }
        storeGraph.logData.value = { logs, phases };
        
        storeGraph.updateGraph();
    } catch (error) {
        console.error("Failed to fetch phase events:", error);
        storeGraph.logData.value = { logs: [], phases: [] };
    } finally {
        storeGraph.loading.value = false;
    }
};

watch(
    [
        () => props.defaultGroup,
        () => props.defaultMetric,
        () => props.defaultLevel
    ],
    () => {
        // do not set defaults if fullscreen or group/metric/level is already set to prevent overwrite
        if (props.fullscreen || form.group || form.metric || form.level) return;

        form.group = props.defaultGroup;
        form.metric = props.defaultMetric;
        form.level = props.defaultLevel;
    },
    { immediate: true }
);

const nodeNames = computed(() =>
    props.jobIds.length ? Object.keys(props.nodes[props.jobIds[0]] || {}) : []
);

watch(
    [() => props.jobIds, () => form.group, () => form.metric, () => form.level, () => form.node],
    () => {
        if (!props.graphId || !form.group || !form.metric || !form.level)
            return;

        if (
            metrics.value?.[form.group] &&
            !(form.metric in metrics.value[form.group])
        )
            return;

        const q = $graphStore.createQuery(
            props.jobIds,
            form.group,
            form.metric,
            form.level as GraphLevel,
            form.node || "",
            false, // deciles
        );

        if (Object.keys(q).length) storeGraph.query.value = q;
    },
    { immediate: true }
);

watchEffect(() => {
    form.group = storeGraph.query.value.group || null;
    form.metric = storeGraph.query.value.metric || null;
    form.level = storeGraph.query.value.level || null;
    form.node = storeGraph.query.value.node || null;
});

const multiNode = computed(
    () => nodeNames.value.length > 1 && form.level != "job"
);

watchEffect(() => {
    if (nodeNames.value.length && !form.node) form.node = nodeNames.value[0];
});

watch(
    () => props.nodes,
    (v) => {
        if (v) storeGraph.nodes.value = { ...v };
    },
    { immediate: true, deep: true }
);

watchEffect(() => {
    if (props.comparisonMode) form.level = "job";
});

// use delay for updating graph colors as useGraph:getColors may otherwise still return old colors
watch(
    theme,
    () => {
        setTimeout(() => {
            storeGraph.updateGraph();
        }, 500);
    },
    { deep: true }
);

const unsubscribe = ref<(() => void) | null>(null);

onMounted(() => {
    unsubscribe.value = $graphStore.cacheClearEvent.on(
        async () => await storeGraph.updateData(true)
    );
});

onUnmounted(() => {
    if (unsubscribe.value !== null) unsubscribe.value();
});
</script>

<style lang="scss" scoped>
@use "~/assets/css/colors.scss" as *;

.statistic-toggle {
    margin-top: 4px !important;
}

.metric-description {
    font-size: 0.75rem;
    color: $font-light;
}

.hint {
    width: 100%;
    white-space: normal;
    color: $font-light;
}

.no-data {
    font-size: 12px;
    color: $font-light;
    font-family: "Source Code Pro";
}

.card {
    // padding is provided by graph
    :deep(.v-card-text) {
        padding: 0 0 10px 0 !important;
    }
    :deep(.v-card-title) {
        padding-top: 0px !important;
    }

    &.rendered {
        overflow: visible;
    }
}

.fullscreen {
    position: absolute;
    right: 10px;
    top: 10px;
}
</style>
