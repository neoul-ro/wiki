---
title: "Git 커밋 히스토리를 한 번에 압축하는 방법"
created: 2026-05-12
updated: 2026-05-12
tags: [how-to, git, history, repository-maintenance]
author: "neoul"
status: "published"
---

# Git 커밋 히스토리를 한 번에 압축하는 방법

커밋이 많이 쌓이면 저장소 히스토리가 길어지고, 오래된 실험 기록이나 작은 수정 커밋까지 계속 따라다닌다. 텍스트 위키처럼 최종 파일 상태가 더 중요한 저장소라면, 필요할 때 과거 커밋을 하나의 snapshot commit으로 압축할 수 있다. 다만 이 작업은 원격 브랜치 히스토리를 다시 쓰는 작업이므로, 혼자 쓰는 저장소이거나 협업자와 합의한 경우에만 해야 한다.

## 결론

개인 publish wiki처럼 현재 파일 상태만 중요하다면 `orphan branch` 방식이 가장 단순하다.

```bash
git checkout --orphan fresh-main
git add -A
git commit -m "Reset history"
git branch -M main
git push --force-with-lease origin main
```

이렇게 하면 현재 파일 상태는 유지하면서 `main` 브랜치의 히스토리를 새 커밋 하나로 교체한다.

## 언제 필요한가

히스토리 압축은 다음 상황에서 쓸 만하다.

- 작은 자동 커밋이 너무 많이 쌓였다.
- 문서 저장소라서 과거 변경 과정보다 현재 publish 상태가 중요하다.
- clone할 때 오래된 히스토리를 받을 필요가 없다.
- 저장소를 공개하기 전에 불필요한 작업 기록을 지우고 싶다.

반대로 협업 중인 코드 저장소라면 신중해야 한다. 다른 사람이 이미 clone한 브랜치를 force push로 바꾸면, 그 사람의 로컬 브랜치와 원격 브랜치가 갈라진다.

## 방법 1: orphan branch로 새 히스토리 만들기

현재 파일 상태만 남기고 과거 커밋을 버리고 싶다면 orphan branch를 만든다.

```bash
git checkout --orphan fresh-main
```

orphan branch는 부모 커밋이 없는 새 브랜치다. 작업 디렉터리의 파일은 그대로 둔 채, Git 히스토리만 끊어진 상태가 된다.

현재 파일을 모두 stage하고 새 snapshot commit을 만든다.

```bash
git add -A
git commit -m "Reset history"
```

브랜치 이름을 다시 `main`으로 바꾼다.

```bash
git branch -M main
```

원격 브랜치를 새 히스토리로 교체한다.

```bash
git push --force-with-lease origin main
```

`--force-with-lease`는 단순 `--force`보다 안전하다. 내가 마지막으로 알고 있던 원격 상태와 실제 원격 상태가 다르면 push를 막아, 다른 사람이 올린 변경을 덮어쓸 가능성을 줄인다.

## 방법 2: rebase로 기존 커밋 squash하기

과거 커밋 메시지나 일부 히스토리를 더 섬세하게 다루고 싶다면 root부터 interactive rebase를 할 수 있다.

```bash
git rebase -i --root
```

편집기가 열리면 첫 커밋은 `pick`으로 두고, 나머지 커밋을 `squash` 또는 `fixup`으로 바꾼다.

```text
pick 1111111 first commit
fixup 2222222 update article
fixup 3333333 fix typo
```

마지막으로 원격에 강제 반영한다.

```bash
git push --force-with-lease origin main
```

이 방식은 히스토리를 보존하면서 정리할 수 있지만, 커밋 수가 많으면 편집이 번거롭다. 단순히 현재 상태 하나만 남길 목적이라면 orphan branch 방식이 더 쉽다.

## clone과 pull이 정말 느려질까

문서 저장소의 작은 텍스트 커밋 수백 개 정도는 보통 clone이나 pull 속도에 큰 영향을 주지 않는다. Git은 object를 압축해서 저장하고 전송하기 때문이다.

다만 아래 상황에서는 히스토리 압축이 체감될 수 있다.

- 큰 바이너리 파일이 과거 커밋에 들어간 적이 있다.
- 이미지, PDF, 데이터 파일이 자주 교체되었다.
- 자동 커밋이 장기간 매우 많이 쌓였다.
- publish 저장소라서 과거 작업 로그를 남길 이유가 없다.

## 주의할 점

히스토리 압축은 되돌리기 어려운 작업처럼 다뤄야 한다.

- 원격 브랜치 히스토리를 다시 쓴다.
- 기존 clone 사용자는 로컬 브랜치를 재설정해야 할 수 있다.
- GitHub에 오래된 object가 즉시 사라진다고 보장할 수는 없다.
- 민감정보를 완전히 제거하려는 목적이라면 별도의 history cleaning 절차가 필요하다.

민감정보가 이미 커밋된 경우에는 단순 squash만으로 충분하지 않을 수 있다. 이때는 `git filter-repo` 같은 도구로 해당 파일이나 문자열을 히스토리 전체에서 제거해야 한다.

## 운영 기준

개인 publish wiki에서는 평소에는 커밋을 계속 쌓아도 괜찮다. 커밋이 너무 많아져서 히스토리가 읽기 어렵거나 저장소를 새로 정리하고 싶을 때만 snapshot commit으로 압축하면 된다.

자동으로 자주 압축하기보다는, 필요할 때 명시적으로 실행하는 편이 안전하다.

## Related

- [[git/_index|Git]]
