<template>
    <div :style="props.style" class="graph-wrapper">
        <v-overlay
            :model-value="true"
            v-if="props.loading"
            contained
            class="align-center justify-center"
        >
            <!-- use v-show on progress as overlay does not correctly hide its slot under on rapid loading state changes -->
            <v-progress-circular
                color="primary"
                v-if="props.loading"
                indeterminate
                size="32"
            ></v-progress-circular>
        </v-overlay>
        <div
            class="graph"
            :id="props.graphId"
            ref="graphRef"
            style="height: 100%"
        ></div>
        <Teleport v-if="mounted" to="body">
            <v-card
                ref="graphCardRef"
                class="hover-info"
                :class="{ visible: hoverVisible }"
                position="absolute"
                :style="`left: ${cursorX + hoverText.leftOffset}px; top: ${
                    cursorY + hoverText.topOffset + pageScrollOffset
                }px;`"
            >
                <div class="header font-italic">{{ hoverText.header }}</div>
                <div class="d-flex gap-20 mb-3">
                    <div
                        v-for="[idx, traces] of hoverText.traces.entries()"
                        class="d-flex justify-space-between"
                    >
                        <div>
                            <div
                                v-for="trace of traces"
                                class="d-flex align-center trace"
                            >
                                <div
                                    :class="
                                        trace?.fullData?.line
                                            ? 'trace-color'
                                            : 'trace-color-marker'
                                    "
                                    :style="`background-color: ${
                                        trace?.fullData?.line?.color ||
                                        trace?.fullData?.marker?.color ||
                                        '#fff'
                                    }`"
                                ></div>
                                <div
                                    class="trace-name flex-grow-1"
                                    :class="{
                                        'font-weight-bold':
                                            trace.curveNumber ===
                                            hoverText.highlight
                                    }"
                                >
                                    {{ trace.data.displayName }}
                                </div>
                                <div
                                    class="trace-value"
                                    :class="{
                                        'font-weight-bold':
                                            trace.curveNumber ===
                                            hoverText.highlight
                                    }"
                                >
                                    {{ roundTo(trace.y) }}
                                </div>
                            </div>
                        </div>
                        <v-divider
                            vertical
                            v-if="
                                hoverText.traces.length > 1 &&
                                idx != hoverText.traces.length - 1
                            "
                        ></v-divider>
                    </div>
                </div>
                <template v-if="hoverText.logs.length">
                    <v-divider class="mb-2" />
                    <div class="d-flex flex-column gap-1 mb-2">
                        <template
                            v-for="[phase, phaseLogs] of hoverLogsByPhase.entries()"
                            :key="phase"
                        >
                            <!-- Phase header -->
                            <div
                                class="d-flex align-center log-phase-header"
                                :style="`color: ${getLogPhaseColor(phase || null)}`"
                            >
                                <div
                                    class="trace-color"
                                    :style="`background-color: ${getLogPhaseColor(phase || null)}`"
                                ></div>
                                <span class="font-weight-bold">{{ phase || 'no phase' }}</span>
                            </div>
                            <!-- Messages within this phase -->
                            <div
                                v-for="log of phaseLogs"
                                :key="log.ts + log.msg"
                                class="d-flex align-center trace log-entry"
                            >
                                <div class="trace-name flex-grow-1 text-medium-emphasis">{{ log.msg }}</div>
                                <div class="trace-value text-medium-emphasis">{{ toDDHHMMSS(log.ts - logStartTime) }}</div>
                            </div>
                        </template>
                    </div>
                </template>
                <div
                    v-show="hoverText.truncated"
                    class="text-medium-emphasis text-caption font-italic"
                >
                    too many traces - legend is truncated
                </div>
            </v-card>
        </Teleport>
    </div>
</template>
<script setup lang="ts">
import { download, roundTo, deepEqual } from "~/utils/misc";
import { toDDHHMMSS } from "~/utils/date";
import { ArrayUtils } from "~/utils/array";
import { useMouse, useElementHover, useScroll } from "@vueuse/core";
import { LEGEND_WIDTH, getLogPhaseColor } from "@/components/graphs/useGraphBase";
import Plotly from "plotly.js-basic-dist-min";
import { Mutex } from "async-mutex";
import type { LogEntry } from "~/types/graph";
import type { StoreGraphReturnDefault } from "~/store/graph";

const plotlySettings = {
    scrollZoom: true,
    displaylogo: false,
    responsive: true,
    doubleClick: "autosize",
    // disable resetScale2d as this resets layout to default settings,
    // select2d and lasso2d are not useful for our graphs and using them makes the current tab crash
    modeBarButtonsToRemove: ["resetScale2d", "select2d", "lasso2d"],
    hovermode: "closest"
};

const { $graphStore } = useNuxtApp();
const graphRef = ref<Plotly.PlotlyHTMLElement | null>(null);
const graphCardRef = ref<HTMLElement | null>(null);

const { y: pageScrollOffset } = useScroll(window);

const emit = defineEmits(["rendered", "relayout"]);

const props = defineProps<{
    graphId: string;
    type: "default" | "roofline";
    relayoutData?: Plotly.PlotRelayoutEvent;
    style?: string;
    loading?: boolean;
}>();

const storeGraph = $graphStore.useStoreGraph(props.graphId, props.type);

const extractor = (event: MouseEvent) =>
    event instanceof Touch ? null : [event.offsetX, event.offsetY];
const { x: cursorX, y: cursorY } = useMouse({
    target: graphRef,
    type: extractor
});

const CHUNK_SIZE = 16;
const CHUNKS = 4;

const mounted = ref(false);
const hoverVisible = ref(false);
const hoverText = reactive<{
    header: string;
    traces: Plotly.PlotData[];
    logs: LogEntry[];
    highlight: number | null;
    leftOffset: number;
    topOffset: number;
    truncated: boolean;
}>({
    header: "",
    traces: [],
    logs: [],
    highlight: null,
    leftOffset: 0,
    topOffset: 0,
    truncated: false
});

const handlersRegistered = ref(false);

const currentGraph = ref({});

// Returns the same color used by useGraphBase.ts for log shapes so the hover is consistent.
// REMOVED – replaced by shared getLogPhaseColor() from useGraphBase.ts

// Builds a map from x-axis index -> log entries for the current graph.
// Uses the same bucketing logic as useGraphBase.ts to match log groups to positions.
const logsByIndex = computed(() => {
    if (props.type !== "default") return new Map<number, LogEntry[]>();
    const sg = storeGraph as StoreGraphReturnDefault;
    let logs = sg.logData.value?.logs || [];
    if (!logs.length) return new Map<number, LogEntry[]>();

    // Apply the same logFilter as useGraph.ts so filtered-out entries don't appear in hover
    const logFilter = sg.modifiers.value.logFilter;
    if (logFilter && logFilter.length > 0) {
        logs = logs.filter((log) => {
            const phase = log.phase ?? 'no phase';
            return logFilter.some(f => phase === f);
        });
    }
    if (!logs.length) return new Map<number, LogEntry[]>();

    const rawEntries = Object.values(sg.raw.value);
    const firstTrace = rawEntries[0]?.traces?.[0];
    if (!firstTrace) return new Map<number, LogEntry[]>();

    const interval = firstTrace.interval || 5;
    // start is a Date – convert to Unix seconds
    const startTimeString = firstTrace.start;

    const startTime = startTimeString
            ? new Date(startTimeString.endsWith('Z') ? startTimeString : startTimeString + 'Z').getTime() / 1000 
            : 0;
            
    // Group by 20-second buckets relative to job start (mirrors useGraphBase.ts)
    const logGroups = new Map<number, LogEntry[]>();
    for (const log of logs) {
        const bucketKey = Math.floor((log.ts - startTime) / 20);
        if (!logGroups.has(bucketKey)) logGroups.set(bucketKey, []);
        logGroups.get(bucketKey)!.push(log);
    }

    const result = new Map<number, LogEntry[]>();
    for (const group of logGroups.values()) {
        const relativeSeconds = group[0].ts - startTime;
        const indexPosition = Math.floor(relativeSeconds / interval);
        result.set(indexPosition, group);
    }
    return result;
});

const logStartTime = computed(() => {
    if (props.type !== "default") return 0;
    const sg = storeGraph as StoreGraphReturnDefault;
    const firstTrace = Object.values(sg.raw.value)[0]?.traces?.[0];
    if (!firstTrace) return 0;
    const startTimeString = firstTrace.start;

    return startTimeString
            ? new Date(startTimeString.endsWith('Z') ? startTimeString : startTimeString + 'Z').getTime() / 1000 
            : 0;
});

// Get the current interval for the graph (used for converting x-values to indices)
const currentInterval = computed(() => {
    if (props.type !== "default") return 5;
    const sg = storeGraph as StoreGraphReturnDefault;
    const firstTrace = Object.values(sg.raw.value)[0]?.traces?.[0];
    return firstTrace?.interval || 5;
});

// Parse a timestamp string in the format "DD HH:MM:SS" or "HH:MM:SS" back to seconds
const parseTimestampToSeconds = (timestamp: unknown): number | null => {
    if (typeof timestamp === 'number') return timestamp;
    if (!timestamp || typeof timestamp !== 'string') return null;
    
    const parts = timestamp.trim().split(' ');
    let days = 0;
    let timeStr = timestamp;
    
    // Check if there's a day component (format: "DD HH:MM:SS")
    if (parts.length === 2) {
        days = parseInt(parts[0], 10) || 0;
        timeStr = parts[1];
    }
    
    const timeParts = timeStr.split(':');
    if (timeParts.length !== 3) return null;
    
    const hours = parseInt(timeParts[0], 10) || 0;
    const minutes = parseInt(timeParts[1], 10) || 0;
    const seconds = parseInt(timeParts[2], 10) || 0;
    
    return days * 86400 + hours * 3600 + minutes * 60 + seconds;
};

// Groups the current hover's log entries by phase for display.
const hoverLogsByPhase = computed(() => {
    const groups = new Map<string, LogEntry[]>();
    for (const log of hoverText.logs) {
        const key = log.phase ?? "";
        if (!groups.has(key)) groups.set(key, []);
        groups.get(key)!.push(log);
    }
    return groups;
});

const getImageBase64 = async (properties: Plotly.ToImgopts) => {
    if (!graphRef.value) return "";
    return await Plotly.toImage(graphRef.value, properties);
};

const exportImg = async (name: string, properties: Plotly.ToImgopts) => {
    download(name, await getImageBase64(properties), "base64");
};

const graphHovered = useElementHover(graphRef);

watch(graphHovered, (v) => {
    if (v) return;
    hoverVisible.value = false;
});

const graphSize = ref<DOMRect | null>(null);

watch(cursorX, (x) => {
    if (!graphSize.value) return;
    if (x >= graphSize.value.width - LEGEND_WIDTH + 5) {
        hoverVisible.value = false;
    }
});

const registerHandlers = () => {
    if (!graphRef.value) return;

    graphRef.value.on(
        "plotly_relayout",
        (eventdata: Plotly.PlotRelayoutEvent) => {
            emit("relayout", eventdata);
        }
    );

    graphRef.value.on("plotly_hover", (data: Plotly.PlotHoverEvent) => {
        let traces = data.points.sort((a, b) => a.curveNumber - b.curveNumber);

        if (!graphCardRef.value || !graphRef.value || !traces.length) {
            hoverVisible.value = false;
            hoverText.logs = [];
            return;
        }

        // TODO updating graph size on each hover is a workaround for graphs within a dialog
        // as they would otherwise have incorrect position/size data
        // if (!graphSize.value) {
        updateGraphSize();
        //     if (!graphSize.value) return;
        // }

        const time = data.points.length ? data.points[0].x : "";
        const unit = traces[0].data.unit || "";

        // Get log entries for the current hover position
        // Use x-value to calculate index instead of pointIndex for robustness during zoom
        let logIndex: number | undefined = undefined;
        
        // First try to calculate from x-value (timestamp string)
        const seconds = parseTimestampToSeconds(time);
        if (seconds !== null) {
            logIndex = Math.floor(seconds / currentInterval.value);
        }
        
        // Fallback to pointIndex if parsing failed
        if (logIndex === undefined) {
            logIndex = data.points[0]?.pointIndex;
        }
        
        if (logIndex !== undefined && logsByIndex.value.has(logIndex)) {
            hoverText.logs = logsByIndex.value.get(logIndex) || [];
        } else {
            hoverText.logs = [];
        }

        hoverText.header = `${time} ${unit ? `[${unit}]` : ""}`;

        if (traces.length > CHUNKS * CHUNK_SIZE) {
            traces = traces.slice(0, CHUNKS * CHUNK_SIZE);
            hoverText.truncated = true;
        } else {
            hoverText.truncated = false;
        }

        hoverText.traces = [
            ...ArrayUtils.chunks(
                traces,
                traces.length > CHUNK_SIZE
                    ? Math.floor(traces.length / CHUNKS)
                    : CHUNK_SIZE
            )
        ];
        
        const hoverSize = graphCardRef.value.$el.getBoundingClientRect();

        if (!graphSize.value) return;

        const leftMargin = graphSize.value.left + 10;

        const clipsRight =
            cursorX.value + hoverSize.width + 10 >= window.innerWidth;
        // rather clip on right side than left side
        // check if hover would clip left if placed on right side
        const clipsLeftOnRightClip =
            cursorX.value - hoverSize.width + graphSize.value.left <= 0;

        hoverText.leftOffset =
            clipsRight && !clipsLeftOnRightClip ? -hoverSize.width : leftMargin;

        const clipsBottom =
            cursorY.value + graphSize.value.top + hoverSize.height >=
            window.innerHeight;

        hoverText.topOffset = graphSize.value.top + 10;

        if (clipsBottom) hoverText.topOffset -= hoverSize.height + 30;

        const y = data.yvals[0];
        let smallestDeviation = null;
        const accountableTraces = hoverText.traces.flat();
        if (accountableTraces.length > 1) {
            for (const p of hoverText.traces.flat()) {
                const deviation = Math.abs(p.y - y);
                if (
                    smallestDeviation === null ||
                    deviation < smallestDeviation
                ) {
                    smallestDeviation = deviation;
                    hoverText.highlight = p.curveNumber;
                }
            }
        } else hoverText.highlight = null;
        hoverVisible.value = true;
    });
};

const graphMounted = computed(() => !!graphRef.value);

const mutex = new Mutex();
watch(
    storeGraph.graph,
    async (g) => {
        // run in mutex as watcher may be triggered multiple times
        await mutex.runExclusive(async () => {
            // nextTick in case graphId is not yet registered as ref
            await nextTick(async () => {
                if (!g || !Object.keys(g).length || !graphRef.value) return;

                // prevent rerender of same graph
                if (deepEqual(g, currentGraph.value)) {
                    emit("rendered");
                    return;
                }

                await Plotly.react(
                    graphRef.value,
                    g.traces.filter((t) => t.visible != "hidden"),
                    g.layout as Partial<Plotly.Layout>,
                    plotlySettings as Partial<Plotly.Config>
                );

                // refrain from using any kind of hooks to register handlers due to differences in CSR/SSR on when the graph div is available
                if (!handlersRegistered.value) {
                    registerHandlers();
                    handlersRegistered.value = true;
                }

                currentGraph.value = deepClone(g);

                emit("rendered");
            });
        });
    },
    { immediate: true, deep: true }
);

watch(
    () => props.relayoutData,
    async (newLayout) => {
        if (!graphRef.value || !newLayout) return;

        // activating Plotlys pan/zoom triggers relayout
        if ("dragmode" in newLayout || "autosize" in newLayout) return;

        const xaxis = graphRef.value.layout.xaxis;

        // graph already using autorange
        if (xaxis.autorange && newLayout["xaxis.autorange"] && xaxis.autorange)
            return;

        // no change in x range
        if (
            "xaxis.range[0]" in newLayout &&
            "xaxis.range[1]" in newLayout &&
            xaxis.range?.[0] == newLayout["xaxis.range[0]"] &&
            xaxis.range?.[1] == newLayout["xaxis.range[1]"]
        )
            return;

        await Plotly.relayout(graphRef.value, newLayout);
        emit("rendered");
    },
    { deep: true }
);

const updateGraphSize = () => {
    if (!graphRef.value) return;
    graphSize.value = graphRef.value.getBoundingClientRect();
};

onMounted(() => {
    updateGraphSize();
    window.addEventListener("resize", updateGraphSize);
    mounted.value = true;
});

onUnmounted(() => {
    window.removeEventListener("resize", updateGraphSize);
});

defineExpose({ exportImg, getImageBase64, graphMounted });
</script>
<style lang="scss" scoped>
.hover-info {
    width: fixed;
    padding: 5px;
    position: absolute !important;
    pointer-events: none;
    opacity: 0;
    visibility: hidden;
    font-family: "Source Code Pro";
    font-size: 0.775rem;
    z-index: 9999;

    &.visible {
        opacity: 1;
        visibility: visible;
    }

    .trace {
        .trace-name {
            width: max-content;
        }
        .trace-value {
            margin-left: 30px;
            min-width: 60px;
        }
        .trace-color {
            margin-right: 5px;
            width: 15px;
            height: 2px;
            border-radius: 2px;
        }
        .trace-color-marker {
            margin-right: 10px;
            // margin-left: 5px;
            width: 5px;
            height: 5px;
            border-radius: 50%;
        }
    }

    .log-phase-header {
        font-size: 0.75rem;
        margin-top: 4px;
        gap: 6px;
        .trace-color {
            flex-shrink: 0;
        }
    }

    .log-entry {
        padding-left: 20px;
    }
}
</style>
