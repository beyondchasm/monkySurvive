<script>
    import { onMount } from "svelte";

    // 미리 정의한 한글 이름 배열
    const monkeyNames = [
        "스가",
        "쿰보",
        "요키치",
        "앤트맨",
        "듀란트",
        "테이텀",
        "부커",
        "브라운",
        "AD",
        "커닝햄",
        "어빙",
        "칼앤타",
        "르브론",
        "팍스",
        "웸비",
        "트레영",
        "미첼",
        "커리",
        "라빈",
        "드로잔",
    ];

    let canvas;
    let ctx;
    let stage = 1;
    const canvasWidth = 800;
    const canvasHeight = 800;

    let monkeys = []; // 현재 살아있는 원숭이
    let deadMonkeys = []; // 탈락한 원숭이 목록
    let bananas = [];
    const numMonkeys = 20;

    let monkeyIdCounter = 1; // 고유번호를 위한 전역 카운터

    // 시뮬레이션 및 게임 모드 관련 변수
    let paused = true; // 초기엔 일시정지 상태 (유저 선택 대기)
    let selectedMonkey = null; // 사용자가 선택한 원숭이 (유저 선택)
    let toggleBtnNm = "재생";
    let playSpeed = 1; // 배속: 1배속 또는 2배속 등

    let simulationInterval;
    let countdownInterval;
    const stageDuration = 5000; // 한 스테이지의 시간 (ms)
    let stageCountdown = stageDuration / 1000; // 초 단위 카운트다운
    let stageEnded = false;
    let simEnded = false; // 시뮬레이션 종료 상태

    // 게임 모드 변수 – 유저가 선택한 원숭이가 마지막 생존자여야 함.
    let guessing = true; // 유저가 아직 선택 전
    let guessResult = null; // null: 미결정, true: 정답, false: 오답
    let eventMsg = "[시스템메시지]";

    // 원숭이 클래스 – stat 범위를 조정하고 이름(name) 속성을 추가
    class Monkey {
        constructor(
            x,
            y,
            health,
            strength,
            intelligence,
            speed,
            effectiveSpeed,
            color,
            name,
            level = 1,
            id,
        ) {
            this.x = x;
            this.y = y;
            this.health = health;
            this.strength = strength;
            this.intelligence = intelligence;
            this.speed = speed;
            this.effectiveSpeed = effectiveSpeed;
            this.color = color;
            this.name = name;
            this.level = level;
            this.id = id;
            this.target = null;
            // 충돌에 의한 impulse를 위한 속도값 초기화
            this.vx = 0;
            this.vy = 0;
        }
        findTarget(bananas) {
            // 지능에 따라 타깃 결정: 지능이 높으면 특별 바나나를 우선, 낮으면 가장 가까운 바나나 선택
            let preferred = bananas.filter((b) => b.special);
            let candidates =
                preferred.length && this.intelligence > 3 ? preferred : bananas;
            let closest = null;
            let closestDist = Infinity;
            candidates.forEach((banana) => {
                let dx = banana.x - this.x;
                let dy = banana.y - this.y;
                let dist = Math.sqrt(dx * dx + dy * dy);
                if (dist < closestDist) {
                    closest = banana;
                    closestDist = dist;
                }
            });
            this.target = closest;
        }
        update() {
            // impulse에 의한 이동 적용
            this.x += this.vx;
            this.y += this.vy;

            // 지능 기반 회피 행동
            const dodgeThreshold = 15;
            if (this.intelligence > 3) {
                let dodgeX = 0,
                    dodgeY = 0;
                monkeys.forEach((other) => {
                    if (other.id !== this.id) {
                        const dx = this.x - other.x;
                        const dy = this.y - other.y;
                        const dist = Math.sqrt(dx * dx + dy * dy);
                        if (dist < dodgeThreshold && dist > 0) {
                            dodgeX += dx / dist;
                            dodgeY += dy / dist;
                        }
                    }
                });
                const dodgeStrength = (this.intelligence - 1) * 0.2;
                this.x += dodgeX * dodgeStrength;
                this.y += dodgeY * dodgeStrength;
            }

            // friction 적용
            const friction = 0.95;
            this.vx *= friction;
            this.vy *= friction;

            if (this.target) {
                let dx = this.target.x - this.x;
                let dy = this.target.y - this.y;
                let dist = Math.sqrt(dx * dx + dy * dy);
                if (dist > 1) {
                    // 기본 이동 벡터
                    let vx = (dx / dist) * this.effectiveSpeed;
                    let vy = (dy / dist) * this.effectiveSpeed;
                    // 이동 거리에 따른 페널티 적용: 이동 거리가 길면 HP가 소모됨 (스테미나 효과)
                    const k = 0.01; // 페널티 강도 (조정 가능)
                    const displacement = Math.sqrt(vx * vx + vy * vy);
                    const penalty = 1 / (1 + k * displacement);
                    vx *= penalty;
                    vy *= penalty;
                    this.x += vx;
                    this.y += vy;
                    // 이동 거리에 비례하여 HP 소모 (스테미나 효과)
                    const staminaCostFactor = 0.0005; // 조정 가능한 소모량
                    this.health -= displacement * staminaCostFactor;
                    if (this.health < 1) this.health = 1; // 최소 HP 1 유지
                }
            } else {
                this.x += (Math.random() - 0.5) * this.speed;
                this.y += (Math.random() - 0.5) * this.speed;
            }

            // 화면 경계 체크 (wrap-around)
            if (this.x < 0) this.x = canvasWidth;
            if (this.x > canvasWidth) this.x = 0;
            if (this.y < 0) this.y = canvasHeight;
            if (this.y > canvasHeight) this.y = 0;
        }
        draw(ctx, isSelected = false) {
            ctx.beginPath();
            const radius = this.strength * 2;
            ctx.arc(this.x, this.y, radius, 0, Math.PI * 2);
            ctx.fillStyle = this.color;
            ctx.fill();

            if (isSelected) {
                ctx.beginPath();
                ctx.arc(this.x, this.y, radius + 4, 0, Math.PI * 2);
                ctx.strokeStyle = "red";
                ctx.lineWidth = 2;
                ctx.stroke();
            }

            ctx.fillStyle = "#fff";
            ctx.font = "12px Arial";
            ctx.fillText(`${this.name}`, this.x - 12, this.y - 12);
        }
    }

    // 바나나 클래스 – 특별 바나나(special) 추가
    class Banana {
        constructor(x, y, special = false) {
            this.x = x;
            this.y = y;
            this.special = special;
        }
        draw(ctx) {
            ctx.beginPath();
            const radius = this.special ? 6 : 4;
            ctx.arc(this.x, this.y, radius, 0, Math.PI * 2);
            ctx.fillStyle = this.special ? "orange" : "yellow";
            ctx.fill();
        }
    }

    // 총 total을 parts 부분으로 랜덤 분할 (각 부분은 0 이상, 합계는 total)
    function randomSplit(total, parts) {
        let cuts = [];
        for (let i = 0; i < parts - 1; i++) {
            cuts.push(Math.random());
        }
        cuts.sort();
        let splits = [];
        let prev = 0;
        for (let i = 0; i < parts - 1; i++) {
            splits.push(cuts[i] - prev);
            prev = cuts[i];
        }
        splits.push(1 - prev);
        return splits.map((s) => s * total);
    }

    // 원숭이 초기화: 고정 총 stat 포인트에서 분배
    function initMonkeys() {
        monkeys = [];
        deadMonkeys = [];
        monkeyIdCounter = 1;

        const fixedTotal = 20; // 각 원숭이의 총 stat 포인트
        const baseHealth = 10;
        const baseStrength = 2;
        const baseIntelligence = 2;
        const baseSpeed = 1;
        const baseTotal =
            baseHealth + baseStrength + baseIntelligence + baseSpeed; // 6
        const remainder = fixedTotal - baseTotal; // 14

        for (let i = 0; i < numMonkeys; i++) {
            const splits = randomSplit(remainder, 4); // 네 스탯에 대해 분배
            let health = baseHealth + splits[0];
            let strength = baseStrength + splits[1];
            let intelligence = baseIntelligence + splits[2];
            let speed = baseSpeed + splits[3];
            const clampedSpeed = Math.min(speed, 5);
            // effectiveSpeed에 제곱근 감쇠를 적용해 극단적 속도 차이가 덜 반영되도록 함
            let effectiveSpeed =
                Math.min(Math.pow(clampedSpeed, 0.7), 5) *
                (0.8 + intelligence / 15);

            let color = `hsl(${Math.floor(Math.random() * 360)}, 100%, 50%)`;
            let name = monkeyNames[i % monkeyNames.length];
            monkeys.push(
                new Monkey(
                    Math.random() * canvasWidth,
                    Math.random() * canvasHeight,
                    health,
                    strength,
                    intelligence,
                    speed,
                    effectiveSpeed,
                    color,
                    name,
                    1,
                    monkeyIdCounter++,
                ),
            );
        }
    }

    // 바나나 생성 – 10% 확률로 특별 바나나 생성
    function initBananas() {
        bananas = [];
        for (let i = 0; i < monkeys.length - 1; i++) {
            let special = Math.random() < 0.1;
            bananas.push(
                new Banana(
                    Math.random() * canvasWidth,
                    Math.random() * canvasHeight,
                    special,
                ),
            );
        }
    }

    // 시뮬레이션 업데이트 (25ms마다 실행, 배속에 따라 조정)
    function updateSimulation() {
        if (paused) return;
        ctx.clearRect(0, 0, canvasWidth, canvasHeight);

        // 원숭이들 개별 업데이트
        monkeys.forEach((monkey) => {
            if (!monkey.target || !bananas.includes(monkey.target)) {
                monkey.findTarget(bananas);
            }
            monkey.update();
            // 바나나와의 상호작용 처리...
            if (monkey.target) {
                let dx = monkey.target.x - monkey.x;
                let dy = monkey.target.y - monkey.y;
                let dist = Math.sqrt(dx * dx + dy * dy);
                if (dist < 8) {
                    if (monkey.target.special) {
                        monkey.health += Math.random() < 0.5 ? 3 : -2;
                        monkey.health = Math.max(1, monkey.health);
                    } else {
                        monkey.health += 1;
                    }
                    bananas = bananas.filter((b) => b !== monkey.target);
                    monkey.target = null;
                }
            }
        });

        // 원숭이들 간의 충돌 처리
        handleCollisions();

        // 바나나 그리기
        bananas.forEach((banana) => banana.draw(ctx));
        // 원숭이 그리기
        monkeys.forEach((monkey) => {
            const isSelected =
                selectedMonkey && monkey.id === selectedMonkey.id;
            monkey.draw(ctx, isSelected);
        });

        // 선택한 원숭이가 살아있는지 실시간 확인 (예측 모드)
        if (
            !guessing &&
            selectedMonkey &&
            !monkeys.find((m) => m.id === selectedMonkey.id)
        ) {
            pauseSimulation();
            guessResult = false;
        }

        if (bananas.length === 0 && !stageEnded) {
            endStage();
        }
        // ★ 실시간 목록 업데이트를 위해 배열 재할당 (객체 내부 변경도 감지하도록)
        monkeys = [...monkeys];
    }

    // 스테이지 종료 로직 – 최종 생존자가 한 마리일 때 결과 판단
    function endStage() {
        if (stageEnded) return;
        stageEnded = true;
        clearInterval(simulationInterval);
        clearInterval(countdownInterval);

        monkeys.forEach((monkey) => {
            monkey.health -= 1;
            if (monkey.health > 0) {
                monkey.level += 1;
            }
        });
        let survivors = monkeys.filter((monkey) => monkey.health > 0);
        let eliminated = monkeys.filter((monkey) => monkey.health <= 0);
        deadMonkeys = deadMonkeys.concat(eliminated);
        monkeys = survivors;

        if (monkeys.length < 2) {
            if (
                monkeys.length === 1 &&
                selectedMonkey &&
                monkeys[0].id === selectedMonkey.id
            ) {
                guessResult = true;
            } else {
                guessResult = false;
            }
            pauseSimulation();
            simEnded = true;
            return;
        } else {
            stage++;
            startStage();
        }
    }

    // 배속 설정 함수
    function setSpeed(speed) {
        playSpeed = speed;
        if (!paused) {
            clearInterval(simulationInterval);
            simulationInterval = setInterval(updateSimulation, 25 / playSpeed);
        }
    }

    function startStage() {
        // 매 스테이지마다 하위순위 원숭이에게 보너스를 적용 (항상 실행)

        if (monkeys.length > 0) {
            let survivors = rankedMonkeys.filter((monkey) => monkey.health > 0);
            // reactive 변수 rankedMonkeys를 이용해 하위순위 원숭이 선택
            let m = survivors[survivors.length - 1];
            // 0: 체력, 1: 힘, 2: 속도, 3: 지능 중 하나 선택
            let statIndex = Math.floor(Math.random() * 4);
            // 변경량: 0.05 ~ 0.25 사이의 양 (필요에 따라 조절)
            let changeAmount = Math.random() * 0.2 + 0.05;
            let statName = "";
            switch (statIndex) {
                case 0:
                    m.health = Math.max(5, m.health + changeAmount);
                    statName = "체력이";
                    break;
                case 1:
                    m.strength = Math.max(1, m.strength + changeAmount);
                    statName = "힘이";
                    break;
                case 2:
                    m.speed = Math.max(0.5, m.speed + changeAmount);
                    const clampedSpeed = Math.min(m.speed, 5);
                    // effectiveSpeed에 제곱근 감쇠를 적용해 극단적 속도 차이가 덜 반영되도록 함
                    m.effectiveSpeed =
                        Math.min(Math.pow(clampedSpeed, 0.7), 5) *
                        (0.8 + m.intelligence / 15);
                    statName = "속도가";
                    break;
                case 3:
                    m.intelligence = Math.max(1, m.intelligence + changeAmount);
                    statName = "지능이";
                    break;
            }
            eventMsg = `[시스템메시지] ${m.name} 원숭이의 ${statName} ${changeAmount.toFixed(3)} 증가했습니다.`;
        } else {
            eventMsg = "[시스템메시지] 이벤트가 없습니다.";
        }

        stageEnded = false;
        stageCountdown = stageDuration / 1000;
        initBananas();
        simulationInterval = setInterval(updateSimulation, 25 / playSpeed);
        countdownInterval = setInterval(() => {
            stageCountdown--;
            if (stageCountdown <= 0) {
                endStage();
            }
        }, 1000 / playSpeed);
    }

    // 캔버스 클릭 처리
    function handleCanvasClick(event) {
        if (paused) return;
        const rect = canvas.getBoundingClientRect();
        const clickX = event.clientX - rect.left;
        const clickY = event.clientY - rect.top;
        for (const monkey of monkeys) {
            let dx = monkey.x - clickX;
            let dy = monkey.y - clickY;
            let dist = Math.sqrt(dx * dx + dy * dy);
            if (dist < 8) {
                selectMonkey(monkey);
                break;
            }
        }
    }

    // 원숭이 충돌 처리 함수 추가
    function handleCollisions() {
        for (let i = 0; i < monkeys.length; i++) {
            for (let j = i + 1; j < monkeys.length; j++) {
                const m1 = monkeys[i];
                const m2 = monkeys[j];
                const dx = m1.x - m2.x;
                const dy = m1.y - m2.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                const radius1 = m1.strength * 2; // 시각적 반지름
                const radius2 = m2.strength * 2;

                if (distance < radius1 + radius2 && distance > 0) {
                    // 겹침 정도 계산
                    const overlap = radius1 + radius2 - distance;
                    // 정규화된 방향 벡터
                    const nx = dx / distance;
                    const ny = dy / distance;

                    // 여기서는 질량을 strength에 비례하도록 정의 (강한 원숭이는 무겁게)
                    const mass1 = m1.strength;
                    const mass2 = m2.strength;
                    const totalMass = mass1 + mass2;

                    // 강한 원숭이가 덜 밀리도록, 보정 계수(factor)를 0.4로 조정
                    const factor = 0.4;
                    const displacement1 =
                        factor * overlap * (mass2 / totalMass);
                    const displacement2 =
                        factor * overlap * (mass1 / totalMass);

                    // 즉각적인 위치 보정으로 겹침 해소
                    m1.x += nx * displacement1;
                    m1.y += ny * displacement1;
                    m2.x -= nx * displacement2;
                    m2.y -= ny * displacement2;

                    // 부드러운 충돌 효과를 위해 속도에도 impulse를 추가 (자연스러운 튕김 효과)
                    m1.vx += nx * displacement1;
                    m1.vy += ny * displacement1;
                    m2.vx -= nx * displacement2;
                    m2.vy -= ny * displacement2;
                }
            }
        }
    }

    function triggerIntelligenceEvent(monkey) {
        // 지능이 일정 수준 이상일 때, 이벤트 발생 확률 상승
        if (monkey.intelligence > 4 && Math.random() < 0.1) {
            // 예시: 체력 소폭 회복
            monkey.health += 2;
            eventMsg = `[시스템메시지] ${monkey.name} 원숭이가 지능으로 위기를 감지하고 체력을 회복했습니다!`;
        }
    }

    function pauseSimulation() {
        paused = true;
        toggleBtnNm = "재생";
        clearInterval(simulationInterval);
        clearInterval(countdownInterval);
    }

    // 결과 모달의 "확인" 버튼 – 게임 재개 (선택 정보는 계속 표시)
    function resumeSimulation() {
        paused = false;
        toggleBtnNm = "일시정지";
        guessResult = null;
        guessing = false;
        startStage();
    }

    function togglePauseAndResume() {
        if (paused) {
            resumeSimulation();
        } else {
            pauseSimulation();
        }
    }

    // 목록에서 원숭이 선택 – 유저가 최종 생존자로 예상하는 원숭이 선택
    function selectMonkey(monkey) {
        if (guessing) {
            selectedMonkey = monkey;
            guessing = false;
            resumeSimulation();
        }
    }

    // 재시작 – 전체 게임 초기화
    function restartSimulation() {
        simEnded = false;
        selectedMonkey = null;
        paused = true;
        stage = 1;
        initMonkeys();
        guessing = true;
        guessResult = null;
    }

    // reactive statement: 전체 원숭이(살아있는+탈락)를 정렬하여 순위 계산
    $: rankedMonkeys = (() => {
        let combined = [...monkeys, ...deadMonkeys];
        combined.sort((a, b) => {
            if (b.health !== a.health) return b.health - a.health;
            if (b.level !== a.level) return b.level - a.level;
            return a.id - b.id;
        });
        for (let i = 0; i < combined.length; i++) {
            combined[i].rank = i + 1;
        }
        return combined;
    })();

    // reactive statement: 선택한 원숭이 순위 갱신
    $: selectedMonkeyRank = selectedMonkey
        ? rankedMonkeys.find((m) => m.id === selectedMonkey.id)?.rank
        : "";

    onMount(() => {
        canvas.width = canvasWidth;
        canvas.height = canvasHeight;
        ctx = canvas.getContext("2d");

        initMonkeys();
        // 게임 모드 시작: 초기에는 시뮬레이션은 정지된 상태에서 유저 선택 대기
        pauseSimulation();
        guessing = true;
        guessResult = null;

        return () => {
            clearInterval(simulationInterval);
            clearInterval(countdownInterval);
        };
    });
</script>

<!-- 상단: 스테이지 번호, 카운트다운, 선택한 원숭이 정보(순위 포함), 배속 버튼, 게임 모드 프롬프트 -->
<div class="header">
    <span class="stage-info">STAGE: {stage}</span>
    <span class="countdown">Countdown: {stageCountdown}초</span>
    {#if selectedMonkey}
        <div class="selected-info">
            <p>
                선택한 원숭이: {selectedMonkey.name} | 순위: {selectedMonkeyRank}
            </p>
        </div>
    {/if}
    <div class="speed-controls">
        {#if !guessing}
            <button on:click={togglePauseAndResume}>{toggleBtnNm}</button>
        {/if}
        <button on:click={() => setSpeed(1)} class:active={playSpeed === 1}
            >1배속</button
        >
        <button on:click={() => setSpeed(2)} class:active={playSpeed === 2}
            >2배속</button
        >
        <button on:click={() => setSpeed(3)} class:active={playSpeed === 3}
            >3배속</button
        >
    </div>
    {#if guessing}
        <div class="game-prompt">
            <p>게임 모드: 마지막까지 살아남을 원숭이를 선택하세요!</p>
            <p>원하는 원숭이를 목록에서 클릭하세요.</p>
        </div>
    {/if}
</div>

<div class="container">
    <!-- 오른쪽: 캔버스 영역 -->
    <div class="canvas-container">
        <span class="system-message">{eventMsg}</span>
        <canvas bind:this={canvas} on:click={handleCanvasClick}></canvas>
    </div>
    <!-- 왼쪽: 원숭이 목록 -->
    <div class="monkey-list">
        <h3>원숭이 순위 목록 (총 {monkeys.length} 마리)</h3>
        {#each rankedMonkeys as monkey (monkey.id)}
            <div
                class="monkey-item {monkey.health <= 0 ? 'inactive' : ''}"
                on:click={() => selectMonkey(monkey)}
            >
                <p>{monkey.name} (ID:{monkey.id})</p>
                <p>순위: {monkey.rank} | HP: {monkey.health.toFixed(3)}</p>
                <p>
                    힘: {monkey.strength.toFixed(2)} | 속도: {monkey.speed.toFixed(
                        2,
                    )} | 지능: {monkey.intelligence.toFixed(2)}
                </p>
            </div>
        {/each}
    </div>
</div>

<!-- 게임 모드 결과 모달 (선택한 원숭이가 최종 생존자인지 결과 표시) -->
{#if !guessing && guessResult !== null}
    <div class="modal-overlay">
        <div class="modal">
            {#if guessResult}
                <h2>축하합니다!</h2>
                <p>선택한 원숭이가 최종 생존자입니다.</p>
            {:else}
                <h2>꽝!</h2>
                <p>선택한 원숭이가 탈락했습니다.</p>
            {/if}
            <div class="modal-buttons">
                <button on:click={restartSimulation}>다시 시작</button>
            </div>
        </div>
    </div>
{/if}

<style>
    .header {
        display: flex;
        flex-wrap: wrap;
        justify-content: space-around;
        align-items: center;
        color: #fff;
        padding: 10px;
        background: #444;
        font-size: 16px;
    }
    .selected-info,
    .game-prompt {
        background: #222;
        padding: 5px 10px;
        border-radius: 4px;
        margin-top: 5px;
    }
    .speed-controls button {
        margin: 5px;
        padding: 8px 12px;
        font-size: 16px;
        border: none;
        background: #666;
        color: #fff;
        border-radius: 4px;
    }
    .speed-controls button.active {
        background: #00aaff;
    }
    .container {
        display: flex;
        flex-wrap: wrap;
        padding: 10px;
        justify-content: center;
    }
    .canvas-container {
        flex: 1;
        min-width: 300px;
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        padding: 10px;
    }
    .system-message {
        color: #fff;
        margin-bottom: 5px;
    }
    canvas {
        background: #333;
        border-radius: 8px;
        margin: 10px;
        max-width: 100%;
    }
    .monkey-list {
        width: 280px;
        padding: 10px;
        background: #555;
        color: #fff;
        font-size: 14px;
        overflow-y: auto;
        max-height: 800px;
        border-radius: 8px;
        margin: 10px;
    }
    .monkey-item {
        padding: 8px;
        margin-bottom: 6px;
        border: 1px solid #777;
        border-radius: 4px;
        cursor: pointer;
        background: #666;
    }
    .monkey-item:hover {
        background: #777;
    }
    .monkey-item.inactive {
        opacity: 0.5;
        background: #333;
        cursor: default;
    }
    .modal-overlay {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.7);
        display: flex;
        justify-content: center;
        align-items: center;
        z-index: 100;
    }
    .modal {
        background: #555;
        padding: 20px;
        border-radius: 8px;
        text-align: center;
        color: #fff;
        max-width: 90%;
    }
    .modal-buttons {
        margin-top: 15px;
    }
    .modal-buttons button {
        margin: 0 10px;
        padding: 10px 15px;
        font-size: 16px;
    }
    :global(body) {
        background: #222;
        margin: 0;
        font-family: Arial, sans-serif;
    }

    /* 모바일 반응형 스타일 */
    @media (max-width: 600px) {
        .container {
            flex-direction: column;
            align-items: center;
        }
        .monkey-list,
        .canvas-container {
            width: 90%;
            margin: 10px 0;
        }
        canvas {
            width: 100%;
            height: auto;
        }
        .header {
            flex-direction: column;
        }
    }
</style>
