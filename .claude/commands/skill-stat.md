스킬별 호출 횟수와 수행 시간 통계를 집계해서 표로 출력합니다.

다음 단계를 순서대로 실행하세요:

1. `logs/skill-usage.jsonl` 파일이 존재하는지 확인합니다. 없으면 "아직 기록된 스킬 사용 내역이 없습니다." 를 출력하고 종료합니다.

2. 아래 명령으로 스킬별 통계를 집계합니다:
   ```
   jq -s 'group_by(.skill) | map({
     skill: .[0].skill,
     count: length,
     min_sec: map(.duration_sec) | min,
     max_sec: map(.duration_sec) | max,
     avg_sec: (map(.duration_sec) | add / length | . * 10 | round / 10)
   }) | sort_by(.count) | reverse' logs/skill-usage.jsonl
   ```

3. 집계 결과를 다음 형식의 마크다운 표로 출력합니다:

   ```
   ## 스킬 사용 통계

   | 스킬 | 호출 횟수 | 최소(초) | 최대(초) | 평균(초) |
   |------|-----------|----------|----------|----------|
   | git-commit | 3 | 10 | 25 | 17.3 |
   | ...        | ...| ...| ...| ...  |

   > 집계 기간: 첫 기록 날짜 ~ 마지막 기록 날짜
   > 총 호출 횟수: N회
   ```

4. 표 아래에 한 줄 인사이트를 추가합니다. 예: "가장 많이 쓴 스킬은 X, 가장 오래 걸린 스킬은 Y입니다."

**주의사항:**
- `jq` 가 없으면 Python3 으로 동일하게 집계합니다:
  ```
  python3 -c "
  import json, collections, math
  lines = open('logs/skill-usage.jsonl').readlines()
  data = [json.loads(l) for l in lines if l.strip()]
  groups = collections.defaultdict(list)
  for d in data: groups[d['skill']].append(d['duration_sec'])
  for skill, times in sorted(groups.items(), key=lambda x: -len(x[1])):
      avg = round(sum(times)/len(times), 1)
      print(f'{skill},{len(times)},{min(times)},{max(times)},{avg}')
  "
  ```
- `$ARGUMENTS` 에 스킬명이 들어오면 해당 스킬만 필터링해서 보여줍니다.
- 집계 기간은 `start` 필드의 최솟값과 최댓값에서 날짜(YYYY-MM-DD)만 추출합니다.

$ARGUMENTS
