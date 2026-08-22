---
title: "BFS 문제를 풀어보고 Claude Code한테 피드백 받기"
date: 2026-04-14
categories: [LLM]
---

백준 1261번(알고스팟) 문제를 직접 풀어본 뒤, Claude Code에게 피드백을 받아본 기록이다.

## 문제와 접근

N×M 미로에서 벽은 이동할 수 없지만 무기로 부술 수 있다. `(0,0)`에서 `(N-1,M-1)`까지
가는 데 필요한 최소 무기(벽을 부수는 횟수) 개수를 구하는 문제다.

먼저 입력 범위부터 따져봤다. N, M이 최대 100이라 전체 셀 수는 최대 10⁴, 웬만한 O(N²)
탐색으로도 여유가 있다. 그리고 미로 값이 0 또는 1뿐이라 `int`로 충분하다는 것도 미리
짚고 들어갔다 — 문제를 풀기 전에 제약 조건부터 훑어보는 습관이 나중에 오버엔지니어링
(예: 불필요하게 `long`을 쓰거나 과도한 최적화를 고민하는 것)을 막아준다.

핵심 판단은 "이게 BFS냐 DFS냐"가 아니라, **가중치가 있는 최단 경로 문제**라는
것이었다. 벽을 부수는 것(비용 1)과 빈 칸으로 이동하는 것(비용 0)이 섞여 있어서,
단순 BFS(모든 간선의 가중치가 1이라고 가정하는 탐색)로는 최소 비용을 보장할 수 없다.
그래서 우선순위 큐를 쓰는 **0-1 BFS류의 다익스트라 변형**으로 접근했다.

```java
private static int bfsService() {
    PriorityQueue<int[]> priorityQueue =
        new PriorityQueue<>(Comparator.comparingInt(value -> value[0]));

    priorityQueue.offer(new int[]{0, 0, 0}); // {비용, row, col}

    while (!priorityQueue.isEmpty()) {
        int[] current = priorityQueue.poll();
        int curCost = current[0], curRow = current[1], curCol = current[2];

        if (curRow == N - 1 && curCol == M - 1) return curCost;

        for (int i = 0; i < 4; i++) {
            int nextRow = curRow + dr[i];
            int nextCol = curCol + dc[i];
            if (nextRow < 0 || nextCol < 0 || nextRow >= N || nextCol >= M) continue;

            int nextCost = curCost + maze[nextRow][nextCol];
            if (nextCost < dist[nextRow][nextCol]) {
                dist[nextRow][nextCol] = nextCost;
                priorityQueue.offer(new int[]{nextCost, nextRow, nextCol});
            }
        }
    }
    return -1;
}
```

우선순위 큐를 쓰면 항상 "지금까지 발견된 것 중 비용이 가장 낮은 경로"부터 꺼내게
되고, `dist` 배열로 이미 더 나은 경로가 기록된 칸은 걸러낸다. 벽(1)과 빈 칸(0)이
섞인 상황에서 이 방식이 단순 BFS보다 확실히 맞는 접근이라는 확신은 있었지만, 머릿속
시뮬레이션만으로는 코드가 실제로 그렇게 동작하는지 완전히 검증하기 어려웠다.

## 왜 AI에게 시뮬레이션을 부탁했나

알고리즘 문제를 풀 때 어려운 지점은 "로직을 설계하는 것"보다 "이 로직이 엣지 케이스에서도
맞는지 손으로 추적하는 것"일 때가 많다. 특히 우선순위 큐처럼 상태가 계속 바뀌는 구조는
머릿속 시뮬레이션의 한계가 금방 온다. Claude Code에게 이 코드를 입력값과 함께 단계별로
시뮬레이션해달라고 요청해서, 큐에 뭐가 들어가고 어떤 순서로 빠지는지, `dist` 배열이
어떻게 갱신되는지를 직접 추적받았다. 이 과정에서 실제로 버그도 하나 잡았다.

## 이 방식이 "그냥 정답 코드 받기"와 다른 이유

Claude Code에게 처음부터 "이 문제 풀어줘"라고 했다면 아마 더 빠르고 정확한 코드를
받았을 것이다. 하지만 그렇게 받은 코드는 내 사고력에 남는 게 없다 — 우선순위 큐를
왜 써야 하는지, 단순 BFS가 왜 이 문제에서는 틀리는지에 대한 판단을 스스로 안 해봤기
때문이다. 반대로 **먼저 스스로 풀고, 그 다음 검증/피드백만 AI에게 맡기는 순서**는
문제를 푸는 능력 자체를 기르는 데 도움이 된다. AI를 학습 도구로 쓸 때 이 순서(직접
풀이 → 검증)가 반대 순서(AI가 먼저 풀이 → 이해 없이 수용)보다 남는 게 많다는 걸
체감한 경험이었다.

토큰 소모에 대한 걱정도 있었는데, 실제로는 적당한 예시 하나로 충분히 버그를 찾고
피드백을 받을 수 있었다 — 전체 테스트 케이스를 다 돌려달라고 할 필요는 없었다.
