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
                    <v-col md="2" sm="6">
                        <v-autocomplete
                            v-model="form.logLevel"
                            label="Log Level"
                            :disabled="
                                !form.metric ||
                                storeGraph.loading.value ||
                                !metricLevels.length ||
                                props.comparisonMode
                            "
                            :items="logLevels"
                            hide-details
                        >
                        </v-autocomplete>
                    </v-col>
                    <v-col md="3" sm="6" v-show="form.logLevel && form.logLevel !== 'None'">
                        <v-autocomplete
                            v-model="logFilterInput"
                            label="Filter Logs"
                            placeholder="Select logs..."
                            :items="availableLogNames"
                            hide-details
                            clearable
                            multiple
                            chips
                            closable-chips
                            density="compact"
                            prepend-inner-icon="$filter"
                            :disabled="storeGraph.loading.value"
                            @update:model-value="debouncedLogFilter"
                            :menu-props="{ maxHeight: 300 }"
                        >
                            <template v-slot:item="{ props, item }">
                                <v-list-item
                                    v-bind="props"
                                    :title="item.raw.title"
                                    :subtitle="item.raw.subtitle"
                                    density="compact"
                                >
                                    <template #prepend="{ isActive }">
                                        <v-checkbox-btn :model-value="isActive" />
                                    </template>
                                    <template #append>
                                        <v-icon
                                            :color="item.raw.color"
                                            size="small"
                                        >mdi-circle</v-icon>
                                    </template>
                                </v-list-item>
                            </template>
                            <template v-slot:chip="{ props, item }">
                                <v-chip
                                    v-bind="props"
                                    :color="item.raw.color"
                                    size="small"
                                    label
                                >
                                    {{ item.raw.title }}
                                </v-chip>
                            </template>
                        </v-autocomplete>
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
                    v-if="
                        Object.keys(metrics.value || {}).length || !props.noData
                    "
                ></ReactiveGraph>
                <!-- graph already displays annotation stating the same issues but graph is never rendered when there are no metics - use this as fallback -->
                <div
                    v-else
                    class="d-flex justify-center no-data"
                    :style="`height: ${props.height}px;`"
                >
                    <div style="margin-top: 160px">no data available</div>
                </div>
            </v-card-text>
        </v-card>
    </div>
</template>

<script setup lang="ts">
import { useGraphForm } from "~/components/graphs/useGraphForm";
import type { NodeMap } from "~/repository/modules/nodes";
import type { GraphLevel } from "~/types/graph";

const { $graphStore } = useNuxtApp();
const graphRef = ref(null);
const { $api } = useNuxtApp();

type RelayoutData = {
    [key: string]: number;
};

interface Props {
    graphId: string;
    jobIds?: number[];
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

const { form, metricGroups, metricLevels, metricItems } = useGraphForm(metrics);

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

        // Preserve the current logLevel when creating a new query
        const currentLogLevel = storeGraph.query.value.logLevel || "None";
        
        const q = $graphStore.createQuery(
            props.jobIds,
            form.group,
            form.metric,
            form.level as GraphLevel,
            form.node || "",
            false, // deciles
            currentLogLevel
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
    // Don't sync logLevel here - it's managed separately by the logLevel watcher
});

watch(
    () => form.logLevel,
    async (newValue, oldValue) => {
        // Skip if no actual change
        if (newValue === oldValue) return;
        
        console.log("logLevel changed:", oldValue, "->", newValue);
        
        if (!newValue || newValue === "None") {
            storeGraph.logs.value = [];
            // Update query with new logLevel and trigger graph update
            const currentQuery = storeGraph.query.value;
            if (currentQuery.logLevel !== newValue) {
                storeGraph.query.value = { ...currentQuery, logLevel: newValue || "None" };
            }
            storeGraph.updateGraph();
            return;
        }

        const jobId = storeGraph.query.value.jobIds?.[0];
        if (!jobId) return;

        storeGraph.loading.value = true;
        try {
            const response = await $api.jobs.get_logs(jobId);
            console.log("Fetched logs:", response);
            
            // Parse logs from API response - format: [[timestamp, message], ...]
            const parsedLogs = (response?.logs || []).map((log: [string, string]) => ({
                timestamp: parseInt(log[0]) / 1000000, // Convert microseconds to seconds
                message: log[1]
            }));
            
            // Set logs first
            storeGraph.logs.value = parsedLogs;
            
            // Update query with new logLevel (this will NOT trigger updateData because logLevel is excluded from that watch)
            const currentQuery = storeGraph.query.value;
            if (currentQuery.logLevel !== newValue) {
                storeGraph.query.value = { ...currentQuery, logLevel: newValue };
            }
            
            // Manually trigger graph update to show the logs
            storeGraph.updateGraph();
            
            console.log("Logs set and graph updated. Query:", storeGraph.query.value);
            console.log("Stored logs:", storeGraph.logs.value);
        } catch (error) {
            console.error("Failed to fetch logs:", error);
            storeGraph.logs.value = [];
        } finally {
            storeGraph.loading.value = false;
        }
    }
);

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
