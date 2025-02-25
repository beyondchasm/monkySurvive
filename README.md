**Monkey Survival**

[몽키서바이벌](https://monky-survive.vercel.app/)
  
**프로젝트 개요**


**Monkey Survival**은 Svelte와 HTML Canvas를 이용해 개발한 실시간 물리 기반 시뮬레이션 게임입니다.

게임은 여러 원숭이가 제한된 자원(바나나)을 두고 이동, 충돌, 상호작용하며 각 스테이지마다 체력(HP), 힘, 지능, 속도 등의 stat이 변화합니다.

특히 사용자는 게임 시작 전에 최종 생존자로 예측할 원숭이를 선택할 수 있으며, 시뮬레이션이 진행된 후 선택한 원숭이의 생존 여부를 확인할 수 있습니다.

  

**주요 특징**

• **고정 stat 분배**

각 원숭이는 고정된 총 stat 포인트(예: 20점)를 기반으로 체력, 힘, 지능, 속도가 랜덤하게 분배됩니다.

이를 통해 모든 원숭이가 동일한 시작 자본을 가지고 경쟁합니다.

• **실시간 물리 시뮬레이션**

• 원숭이들은 Canvas를 이용해 실시간으로 움직이며, 충돌 및 회피 행동을 합니다.

• 바나나를 먹으면 체력이 회복되고, 충돌이나 긴 이동으로 인해 스테미너(HP)가 소모됩니다.

• 각 프레임마다 원숭이의 HP가 업데이트되어 목록에서 소수점 세 자리까지 실시간으로 표시됩니다.

• **예측 게임 모드**

게임 시작 전에 사용자는 마지막 생존자가 될 원숭이를 선택할 수 있습니다.

시뮬레이션이 진행되면서 선택한 원숭이의 생존 여부에 따라 승리 또는 패배 결과를 확인할 수 있습니다.

• **랜덤 이벤트와 밸런스 조정**

매 스테이지마다 하위순위 원숭이에게 작은 보너스가 자동으로 적용됩니다.

예를 들어, 체력, 힘, 속도, 지능 중 한 가지 stat이 무작위로 선택되어 ±0.05~0.25 만큼 증가하며, 이를 시스템 메시지로 표시합니다.

이를 통해 하위 순위 원숭이도 역전의 기회를 갖게 되어 게임의 밸런스를 맞춥니다.

• **모바일 반응형 UI**

게임 인터페이스는 데스크탑과 모바일 모두에서 최적화되어 표시됩니다.

원숭이 목록, 캔버스 영역, 시스템 메시지 등이 깔끔하게 구성되어 사용자에게 직관적인 경험을 제공합니다.

  

**기술 스택**

• **Frontend Framework:** Svelte

• **Rendering:** HTML Canvas

• **언어:** JavaScript, HTML, CSS

  

**게임 플레이**

1. **시작 화면**

게임 시작 전, 원숭이 목록이 표시되며 사용자는 마지막 생존자로 예측할 원숭이를 선택할 수 있습니다.

선택 후, 시뮬레이션이 시작되고 스테이지마다 원숭이들의 이동, 충돌, 상호작용이 실시간으로 진행됩니다.

2. **실시간 시뮬레이션**

• 원숭이들은 바나나를 찾아 이동하며, 충돌 시 물리적인 반발 효과를 받습니다.

• 이동 거리가 길면 스테미너(HP)가 소모되어, 목록에서 체력 변동을 소수점 세 자리까지 실시간으로 확인할 수 있습니다.

• 각 스테이지가 종료되면 하위순위 원숭이에게 소폭의 stat 보너스가 적용됩니다.

1. **결과 확인**

시뮬레이션이 종료되면, 사용자가 예측한 원숭이가 최종 생존자인지 결과 모달로 확인할 수 있습니다.

  

**코드 개요**

  

프로젝트의 주요 코드 구성은 다음과 같습니다:

• **원숭이( Monkey ) 클래스**

• stat 초기화 및 랜덤 분배

• 타깃( 바나나 ) 탐색, 이동, 충돌 회피, HP 소모(스테미너 효과)

• 실시간으로 Canvas에 그리기

• **바나나( Banana ) 클래스**

• 일반 바나나와 특별 바나나의 구분 및 그리기

• **시뮬레이션 로직**

• 매 프레임마다 업데이트되는 원숭이의 위치, 충돌 처리, 바나나 상호작용 등

• 스테이지 종료 및 재시작 로직

• 사용자가 선택한 원숭이의 생존 여부를 확인하는 예측 게임 모드

• **실시간 UI 업데이트**

• Svelte의 reactive statement를 이용하여 원숭이 목록의 체력(HP) 및 기타 stat을 실시간으로 갱신

  

**예제 코드**

```
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
                m.effectiveSpeed = Math.min(Math.pow(clampedSpeed, 0.7), 5) * (0.8 + m.intelligence / 10);
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
```

**결론**

  

**Monkey Survival**은 Svelte와 Canvas를 활용한 실시간 물리 시뮬레이션 게임입니다.

각 원숭이의 stat 분배, 스테미너(HP 소모) 시스템, 랜덤 이벤트 및 예측 게임 모드 등이 결합되어, 플레이어에게 역동적이고 예측하기 어려운 게임 플레이를 제공합니다.

프로젝트의 코드를 기반으로 다양한 밸런스 조정과 기능 확장이 가능하므로, 여러분의 아이디어를 추가하여 더욱 발전시킬 수 있습니다.

이 문서를 통해 프로젝트의 전체적인 개념과 구현 방식을 이해하고, 필요에 따라 커스터마이징할 수 있기를 바랍니다.