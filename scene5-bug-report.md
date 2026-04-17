# [Scene 5] 작업 모드 - 중단 버튼 클릭 시 앱 Freezing 및 중단/제외 기능 미작동

**레이블:** `bug`, `high-priority`, `flutter`, `scene-5`

---

## 개요

작업 모드(Scene 5)에서 중단 버튼 클릭 시 앱이 비활성화(Freezing)되며,
중단 및 제외 기능이 정상적으로 동작하지 않음.

---

## 현재 동작 (Current Behavior)

### 문제 1 - 중단 버튼 클릭 시 앱 Freezing

1. 작업 진행 중 **중단** 버튼 클릭
2. 중단 사유 다이얼로그 표시됨
3. 사유 입력 후 확인 클릭
4. `_isPerformingWork = true` 로 전환되어 **모든 버튼이 비활성화**
5. 내부적으로 `_collectDeviceInfo()` → API 호출 순서로 진행되나,
   타임아웃 없이 무한 대기 상태 진입 가능
6. 결과: 완료/중단/제외 버튼 전체가 응답 불가 상태 → **앱 Freezing**

### 문제 2 - 중단 기능 미작동

- 사진 첨부 없는 경우: `_changeStatus('paused')` 호출 → `_selectedTask`가
  paused 상태로 업데이트는 되나, UI 버튼 상태가 기대와 다르게 동작
- 사진 첨부 있는 경우: `_performWork('paused')` 호출 → 성공 시
  `_selectedTask = null` 로 초기화되어 **선택된 작업이 사라짐**
- 두 경로의 결과 동작이 불일치

### 문제 3 - 제외(요청) 기능 미작동

- 조종자(worker) 역할: `_requestExclusion()` 호출 후 API 성공 시에도
  로컬 `TaskModel` 상태가 업데이트되지 않아
  UI상 여전히 `in_progress` 상태로 표시됨
- 중복 요청 방지 로직 없어 버튼 연속 클릭 시 API가 반복 호출됨

---

## 기대 동작 (Expected Behavior)

### 문제 1 - Freezing 해결

- 중단 버튼 클릭 후 로딩 스피너가 표시되는 동안에도 **앱은 응답 가능 상태** 유지
- `_collectDeviceInfo()` 에 타임아웃(예: 3초) 적용 → 초과 시 `{'platform': 'unknown'}` 반환하고 계속 진행
- 네트워크 오류/타임아웃 발생 시 에러 메시지 표시 후 `_isPerformingWork = false` 로 복구

### 문제 2 - 중단 기능 정상화

- 중단 성공 시 `_selectedTask`의 상태가 `paused`로 업데이트되어 패널에 반영됨
- 사진 유무와 관계없이(`_changeStatus` / `_performWork` 두 경로 모두)
  동일하게 `_selectedTask = task.copyWith(status: 'paused')` 유지

### 문제 3 - 제외 요청 기능 정상화

- 조종자의 제외 요청 성공 후 해당 `TaskModel`에 "제외 요청 중" 상태 반영
- 요청 진행 중 버튼 비활성화로 중복 요청 방지
- "제외 요청이 관리자에게 전송되었습니다" 스낵바 표시 후 버튼 재활성화

---

## 재현 경로 (Steps to Reproduce)

**Freezing:**
1. 작업 모드 진입 → 필지 선택 → 작업 시작
2. 하단 패널에서 **중단** 버튼 클릭
3. 중단 사유 입력 → 사진 첨부 후 확인
4. → 모든 버튼 비활성화, 앱 무응답

**제외 요청 미반영:**
1. 조종자(worker) 계정으로 로그인
2. 작업 모드 → 필지 선택 → 작업 시작
3. 하단 패널에서 **제외 요청** 클릭 → 사유 입력 → 확인
4. → 스낵바 표시되나 UI는 여전히 `in_progress` 상태 표시

---

## 원인 파악 (Root Cause)

| # | 파일 | 라인 | 원인 |
|---|------|------|------|
| 1 | `work_mode_panels.dart` | L593–619 | `_collectDeviceInfo()` 타임아웃 없음 → 무한 대기 가능 |
| 2 | `work_mode_panels.dart` | L624, L732 | `_isPerformingWork = true` 이후 모든 버튼 `onPressed: null` → UI 전체 차단 |
| 3 | `work_mode_panels.dart` | L647–648 | `_performWork('paused')` 성공 시 `_selectedTask = null` → 완료/제외와 동일하게 처리됨 |
| 4 | `work_mode_panels.dart` | L878–919 | `_requestExclusion()` 성공 후 로컬 상태 미업데이트 |
| 5 | `selected_task_panel.dart` | L277, L294 | 버튼 `onPressed: isPerformingWork ? null : callback` → 작업 중 모든 제어 불가 |

---

## 수정 방향

```dart
// 1. _collectDeviceInfo 타임아웃 추가
final deviceInfo = await _collectDeviceInfo()
    .timeout(const Duration(seconds: 3), onTimeout: () => {'platform': 'unknown'});

// 2. _performWork 내 paused 처리 수정 (null 제거)
setState(() {
  if (status == 'completed' || status == 'excluded') {
    _selectedTask = null;
  } else {
    _selectedTask = task.copyWith(status: status); // paused도 유지
  }
});

// 3. _requestExclusion 성공 후 로컬 상태 업데이트 추가
setState(() {
  _selectedTask = task.copyWith(exclusionRequestPending: true);
});
```

---

## 관련 파일

- `lib/presentation/work_mode/screens/work_mode_panels.dart`
- `lib/presentation/work_mode/widgets/selected_task_panel.dart`
- `lib/data/repositories/task_audit_ext.dart`
