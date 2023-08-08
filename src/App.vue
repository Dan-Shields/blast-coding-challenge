<template>
    <button type="button" @click="open()">Open new log file</button>
    <button type="button" @click="close" v-if="log">
        Close {{ files?.[0].name }}
    </button>

    <div v-show="log">
        <pre class="overflow-scroll max-h-[300px]">{{ log }}</pre>

        <div v-show="matchData">
            <h3>Match Metadata</h3>
            <pre>{{ matchMetaString }}</pre>
            <h3>Rounds selector</h3>
            <div ref="roundSlider" class="mx-4" />
            <div class="mt-20">{{ aggregatedData }}</div>
            <div class="w-[1024px] h-[1024px] relative">
                <canvas
                    class="absolute z-10"
                    width="1024"
                    height="1024"
                    ref="heatmapContainer"
                />
                <img class="absolute" src="./assets/maps/de_nuke/radar.png" />
            </div>
        </div>
    </div>
</template>

<script lang="ts" setup>
import { useFileDialog } from "@vueuse/core"
import { computed, nextTick, onMounted, ref, watch } from "vue"
import noUiSlider, { PipsMode } from "nouislider"
import "nouislider/dist/nouislider.css"

import simpleheat from "simpleheat"

const roundSlider = ref<HTMLElement>()
const heatmapContainer = ref<HTMLElement>()

const log = ref("")
const selectedRoundNums = ref()
let heat

onMounted(() => {
    if (!heatmapContainer.value) return

    var heat = simpleheat(heatmapContainer.value)

    heat.data([
        [500, 500, 0.5],
        [100, 100, 1],
    ])

    heat.draw()
    console.log(heat)
})

const reader = new FileReader()

reader.addEventListener(
    "load",
    () => {
        if (typeof reader.result !== "string") {
            log.value = "Could not read text file as string"
            return
        } else {
            processLog(reader.result)
        }
    },
    false
)

const { files, open, reset, onChange } = useFileDialog({
    multiple: false,
    accept: ".txt",
})

onChange((files) => {
    if (!files?.[0]) {
        log.value = ""
    } else {
        reader.readAsText(files[0])
    }
})

function close() {
    reset()
    reader.abort()
    log.value = ""
}

type Side = "t" | "ct"

interface Kill {
    killerSide: Side
    attackLocation: number[]
    deathLocation: number[]
    ts: number
}

interface Player {
    name: string
    id: string
}

interface Team {
    name: string
    startSide: Side
    players: Player[]
}

interface Round {
    winner: Side
    kills: Kill[]
    startTs: number
    duration: number
    endScore: number[]
}

interface MatchData {
    meta: Partial<{
        startTs: number
        endTs: number
        htTs: number
        finalScore: number[]
        map: string
    }>
    rounds: Round[]
    team1: Team
    team2: Team
}

const matchData = ref<MatchData>()

// derive some data from match metadata and format into a nice string
const matchMetaString = computed(() => {
    if (!matchData.value) return

    let meta = {
        ...matchData.value.meta,
        startTs: new Date(matchData.value.meta.startTs).toString(),
        endTs: new Date(matchData.value.meta.endTs).toString(),
        roundCount:
            matchData.value.meta.finalScore[0] +
            matchData.value.meta.finalScore[1],
        matchup: `${matchData.value.team1.name} (started ${matchData.value.team1.startSide}) vs ${matchData.value.team2.name}`,
        finalScore: matchData.value.meta.finalScore.join("-"),
    }

    return Object.keys(meta)
        .map((key) => `${key}:\t ${meta[key]}`)
        .join("\n")
})

// return line ts as epoch time from a line string
// example log ts format: "11/28/2021 - 20:43:11"
// needs to be in format: "YYYY-MM-DDTHH:mm:ss"
function getLineTs(line: string): number {
    let year = line.slice(6, 10)
    let month = line.slice(0, 2)
    let day = line.slice(3, 5)

    let hour = line.slice(13, 15)
    let minute = line.slice(16, 18)
    let second = line.slice(19, 21)
    let string = `${year}-${month}-${day}T${hour}:${minute}:${second}`
    return new Date(string).getTime()
}

// populate matchData from a stringified log
function processLog(rawLog: string) {
    log.value = rawLog
    const lines = rawLog.split("\n")

    const _matchData: any = {
        meta: {},
        rounds: [],
        team1: {},
        team2: {},
    }

    let roundNum = 1
    let currentRound: any = {
        kills: [],
    }

    lines.forEach((line, lineIndex) => {
        if (line.includes('"Match_Start"')) {
            _matchData.meta.startTs = getLineTs(line)

            _matchData.meta.map = line.split(" ")[7].slice(1, -1)

            // match (re)-started, reset stats
            roundNum = 1
            _matchData.rounds = []
            _matchData.team1 = {
                name: lines[lineIndex + 1].split(" ").slice(6).join(" "),
                startSide: lines[lineIndex + 1].split(" ")[5].slice(1, -2),
            }
            _matchData.team2 = {
                name: lines[lineIndex + 2].split(" ").slice(6).join(" "),
                startSide: lines[lineIndex + 2].split(" ")[5].slice(1, -2),
            }
            currentRound = { kills: [] }
            return
        }

        if (line.includes(": Game Over: ")) {
            _matchData.meta.endTs = getLineTs(line)

            let scoreString = line.split(" ")[9]

            _matchData.meta.finalScore = scoreString
                .split(":")
                .map((num) => parseInt(num))
            return
        }

        const killInfo =
            /"(?<killer>.*)" \[(?<killer_location>-?[0-9]+ -?[0-9]+ -?[0-9]+)\] killed "(?<victim>.*)" \[(?<victim_location>-?[0-9]+ -?[0-9]+ -?[0-9]+)\] with "(?<weapon>.*)"/.exec(
                line
            )

        if (killInfo) {
            // this line is a kill!
            currentRound.kills.push({
                killerSide: killInfo.groups["killer"]
                    .split("<")
                    .pop()
                    .slice(0, -1),
                attackLocation: killInfo.groups["killer_location"].split(" "),
                deathLocation: killInfo.groups["victim_location"].split(" "),
                ts: getLineTs(line),
            })
            return
        }

        if (line.includes("Round_Start")) {
            currentRound.startTs = getLineTs(line)
            return
        }

        if (line.includes("Round_End")) {
            const endScore = lines[lineIndex - 1].split(" ")[5].split(":")
            currentRound.endScore = endScore
            currentRound.duration =
                (getLineTs(line) - currentRound.startTs) / 1000
            _matchData.rounds.push(currentRound)
            currentRound = { kills: [] }
            roundNum++
            return
        }
    })

    // TODO: check all necessary data has been located

    matchData.value = _matchData

    nextTick(() => {
        if (!roundSlider.value) return
        ;(roundSlider.value as any)?.noUiSlider?.destroy()

        const numRounds =
            matchData.value.meta.finalScore[0] +
            matchData.value.meta.finalScore[1]

        noUiSlider.create(roundSlider.value, {
            start: [1, numRounds],
            connect: true,
            step: 1,
            range: {
                min: 1,
                max: numRounds,
            },
            pips: {
                mode: PipsMode.Steps,
            },
            behaviour: "drag",
            format: {
                from: function (formattedValue) {
                    return Number(formattedValue)
                },
                to: function (numericValue) {
                    return Math.round(numericValue)
                },
            },
        })
        ;(roundSlider.value as any)?.noUiSlider.on("update", (e) => {
            selectedRoundNums.value = e
        })
    })
}

// generate cumulative/aggregated data from selected rounds
const aggregatedData = computed(() => {
    if (!matchData.value || !selectedRoundNums.value) return null

    const rounds = matchData.value.rounds.slice(
        selectedRoundNums.value[0],
        selectedRoundNums.value[1] + 1
    )
    const totals = rounds.reduce(
        (totals, curr) => {
            totals.kills += curr.kills.length
            totals.duration += curr.duration
            return totals
        },
        { kills: 0, duration: 0 }
    )
    const averages = {}
    for (const [key, value] of Object.entries(totals)) {
        averages[key] = parseFloat(Number(value / rounds.length).toPrecision(4))
    }
    return {
        totals,
        averages,
    }
})

// fetch the default log on page load
fetch("/match-log.txt").then(async (res) => {
    processLog(await res.text())
})
</script>

<style>
.noUi-marker {
    background: none !important;
}
.noUi-value-sub {
    font-size: inherit;
}
</style>
