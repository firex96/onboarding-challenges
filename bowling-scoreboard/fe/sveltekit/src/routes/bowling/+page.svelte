<script lang="ts">
	type Frame = {
		counts: (number | null)[];
		score: number | null;
        isStrike: boolean;
        isSpare: boolean;
	};

	type Player = {
        id: number;
		name: string;
		frames: Frame[];
		total: number | null;
	};

    let current = $state({ playerIndex: 0, frameIndex: 0, countIndex: 0, availableCount: -1 });

    // 빈 프레임 생성 헬퍼 (1~9프레임 2구, 10프레임 3구)
    function createEmptyFrames(): Frame[] {
        return Array.from({ length: 10 }, (_, i) => ({
            counts: i === 9 ? [null, null, null] : [null, null],
            score: null,
            isStrike: false,
            isSpare: false,
        }));
    }

    let players = $state<Player[]>([
        { id: 1, name: 'Player1', frames: createEmptyFrames(), total: null },
        { id: 2, name: 'Player2', frames: createEmptyFrames(), total: null },
    ]);

    function setCount(count: number | null) {
        if (count !== null && (count < 0 || count > 10))
            return;

        const player = players[current.playerIndex];
        const frame = player.frames[current.frameIndex];

        if(count == null && frame.counts[current.countIndex] == null) {
            moveToNextCount(-1);
            setCount(null);
        } else {
            frame.counts[current.countIndex] = count;

            if (count !== null) {
                moveToNextCount(1);
            } else {
                moveToNextCount(-1);
            }
        }
    }

    function moveToNextCount(direction: number) {
        const frame = players[current.playerIndex].frames[current.frameIndex];
        const isLastFrame = current.frameIndex == 9;

        if(direction !== -1 && direction !== 1)
            return;

        if(direction == 1) {

            if (current.countIndex + 1 < frame.counts.length) {
                if(current.countIndex == 0) {
                    if(frame.counts[0] == 10) {
                        frame.isStrike = true;

                        if (!isLastFrame)
                            current.frameIndex += 1;
                        else {
                            current.countIndex += 1;
                        }
                    } else {
                        current.availableCount = 10 - (frame.counts[0] == null ? 0 : frame.counts[0]);
                        current.countIndex += 1;
                    }
                } else {
                    if((frame.counts[0] == null ? 0 : frame.counts[0]) + (frame.counts[1] == null ? 0 : frame.counts[1]) == 10) {
                        frame.isSpare = true;

                        if(!isLastFrame)
                            current.frameIndex += 1;
                        else
                            current.countIndex += 1;
                    } else {
                        if(!isLastFrame)
                            current.frameIndex += 1;
                        else {
                            calculateScore();

                            current.playerIndex += 1;
                            current.frameIndex = 0;
                            current.countIndex = 0;
                        }
                    }
                }
            } else{
                calculateScore();

                if (current.frameIndex + 1 < 10) {
                    current.frameIndex += 1;
                    current.countIndex = 0;
                    current.availableCount = -1;
                } else {
                    if (current.playerIndex + 1 < players.length) {
                        current.playerIndex += 1;
                        current.frameIndex = 0;
                        current.countIndex = 0;
                    }
                }
            }
        } else {
            if (current.countIndex - 1 >= 0) {
                calculateScore();

                current.countIndex -= 1;
                current.availableCount = 10 - (frame.counts[0] == null ? 0 : frame.counts[0]);
            } else {
                if(current.frameIndex - 1 >= 0) {
                    current.frameIndex -= 1;

                    if(isLastFrame) {
                        let lastFrame = players[current.playerIndex].frames[current.frameIndex];
                        current.countIndex = (lastFrame.isStrike || lastFrame.isSpare) ? 3 : 2;
                    }

                    current.availableCount = -1;
                } else {
                    if (current.playerIndex - 1 >= 0) {
                        calculateScore();

                        current.playerIndex -= 1;
                        current.frameIndex = 9;
                        current.countIndex = 3;
                    }
                }
            }
        }
    }

    function calculateScore() {
        const thisFrame = players[current.playerIndex].frames[current.frameIndex];
        const oneFramesBack = current.frameIndex > 0 ? players[current.playerIndex].frames[current.frameIndex] : null;
        const twoFramesBack = current.frameIndex > 1 ? players[current.playerIndex].frames[current.frameIndex] : null;
        const isLastFrame = current.frameIndex == 9;
        const firstThisFrameCount = thisFrame.counts[0] == null ? 0 : thisFrame.counts[0];
        const secondThisFrameCount = thisFrame.counts[1] == null ? 0 : thisFrame.counts[1];

        if(twoFramesBack != null) {
            if(twoFramesBack?.isStrike) {
                if(oneFramesBack?.isStrike)
                    twoFramesBack.score = 20 + (thisFrame.counts[0] == null ? 0 : thisFrame.counts[0]);
            }
        }

        if(oneFramesBack != null) {
            if(oneFramesBack.isStrike) {
                if(isLastFrame)
                    oneFramesBack.score = 10 + firstThisFrameCount + secondThisFrameCount;
                else if(!thisFrame.isStrike)
                    oneFramesBack.score = 10 + firstThisFrameCount + secondThisFrameCount;
            } else if(oneFramesBack.isSpare) {
                if(!thisFrame.isStrike)
                    oneFramesBack.score = 10 + firstThisFrameCount + secondThisFrameCount;
                else
                    oneFramesBack.score = 20;
            }
        }

        if(!thisFrame.isStrike && !thisFrame.isSpare) {
            thisFrame.score = firstThisFrameCount + secondThisFrameCount;
        }

        const player = players[current.playerIndex];
        player.frames.forEach(frame => {
            player.total = frame.score;
        })
    }
</script>

<svelte:head>
	<title>Bowling Scoreboard</title>
	<meta name="description" content="A Wordle clone written in SvelteKit" />
</svelte:head>

<h1 class="visually-hidden">Bowling Scoreboard</h1>

<table class="scoreboard">
    <thead>
        <tr>
            <th>선수</th>
            {#each Array.from(Array(9).keys()) as frameNum (frameNum)}
                <th colspan="2">{frameNum + 1}</th>
            {/each}
            <th colspan="3">10</th>
            <th>총점</th>
        </tr>
    </thead>

    <tbody>
        {#each players as player (player.id)}
            <!-- 1행: 프레임별 투구 기록 (1~9프레임 2칸, 10프레임 3칸) -->
            <tr class="counts-row">
                <td rowspan="2" class="player-name">{player.name}</td>
                {#each player.frames as frame, frameNumber (frameNumber)}
                    {#each frame.counts as count, countNumber (countNumber)}
                        <td class="count-cell">{count}</td>
                    {/each}
                {/each}
                <td rowspan="2" class="total-score">{player.total}</td>
            </tr>

            <!-- 2행: 프레임별 누적 점수 -->
            <tr class="score-row">
                {#each player.frames as frame, frameNumber (frameNumber)}
                    <td colspan={frameNumber === 9 ? 3 : 2} class="frame-score">
                        {frame.score ?? ''}
                    </td>
                {/each}
            </tr>
        {/each}
    </tbody>
</table>

<div class="controls">
    <div class="status">
        입력 위치: <strong>{players[current.playerIndex].name}</strong> -
        {current.frameIndex + 1}프레임 {current.countIndex + 1}구
    </div>

    <div class="pin-buttons">
        {#each Array.from(Array(11).keys()) as pin(pin)}
            <button type="button" onclick={() => setCount(pin)}
                    disabled="{current.availableCount !== -1 && pin > current.availableCount}"
            >{pin}</button>
        {/each}
        <button type="button" class="clear-button" onclick={() => setCount(null)}>지우기</button>
    </div>
</div>

<style>
	.scoreboard {
		border-collapse: collapse;
		text-align: center;
		font-family: sans-serif;
	}

	.scoreboard th,
	.scoreboard td {
		border: 1px solid #333;
		padding: 4px;
	}

	.player-name {
		min-width: 80px;
		font-weight: bold;
	}

	.count-cell {
		width: 28px;
		height: 24px;
		font-size: 0.9rem;
	}

	.frame-score {
		height: 32px;
		font-weight: bold;
	}

	.total-score {
		min-width: 60px;
		font-size: 1.1rem;
		font-weight: bold;
	}

    .controls {
        margin-bottom: 16px;
        display: flex;
        flex-direction: column;
        gap: 8px;
        font-family: sans-serif;
    }

    .pin-buttons {
        display: flex;
        gap: 4px;
        flex-wrap: wrap;
    }

    .pin-buttons button {
        min-width: 40px;
        height: 40px;
        padding: 0 10px;
        border: 1px solid var(--color-theme-2);
        border-radius: 6px;
        background: white;
        color: var(--color-theme-2);
        font-size: 1rem;
        font-weight: bold;
        cursor: pointer;
        box-shadow: 0 2px 0 rgba(0, 0, 0, 0.15);
        transition:
            background-color 0.1s,
            color 0.1s;
    }

    .pin-buttons button:hover {
        background: var(--color-theme-2);
        color: white;
    }

    .pin-buttons button:active {
        transform: translateY(2px);
        box-shadow: none;
    }

    .pin-buttons button:focus-visible {
        outline: 2px solid var(--color-theme-1);
        outline-offset: 2px;
    }

    .pin-buttons button:disabled {
        opacity: 0.35;
        cursor: not-allowed;
        box-shadow: none;
    }

    .pin-buttons button:disabled:hover {
        background: white;
        color: var(--color-theme-2);
    }

    .pin-buttons .clear-button {
        margin-left: 8px;
        border-color: var(--color-theme-1);
        color: var(--color-theme-1);
    }

    .pin-buttons .clear-button:hover {
        background: var(--color-theme-1);
        color: white;
    }
</style>
