---
title: "GitHub 403 인증 오류: 여러 계정에서 SSH 키를 분리해 해결하기"
created: 2026-05-12
updated: 2026-05-12
tags: [troubleshooting, git, github, authentication]
author: "neoul"
status: "published"
---

# GitHub 403 인증 오류: 여러 GitHub 계정에서 SSH 키를 분리해 해결하기

GitHub 저장소에 push할 때 `Permission denied to <다른 계정>` 또는 `403`이 나오면, 현재 Git 작업이 대상 저장소 권한을 가진 계정이 아니라 다른 GitHub 계정으로 인증되고 있다는 뜻이다. 여러 GitHub 계정을 한 Mac에서 쓴다면 계정별 SSH key를 만들고, `~/.ssh/config`의 Host alias로 저장소 remote를 분리하는 방식이 가장 명확하다.

## 증상

다음처럼 `TARGET_OWNER/REPO.git`에 push하려 했지만 GitHub가 `OLD_ACCOUNT` 계정으로 인증해 거절했다.

```bash
git remote add origin git@github.com:TARGET_OWNER/REPO.git
git push -u origin main
```

출력은 다음 형태다.

```text
error: remote origin already exists.
remote: Permission to TARGET_OWNER/REPO.git denied to OLD_ACCOUNT.
fatal: unable to access 'https://github.com/TARGET_OWNER/REPO.git/': The requested URL returned error: 403
```

핵심 단서는 두 가지다.

- `remote origin already exists.`: 이미 `origin` remote가 있어서 새 remote가 추가되지 않았다.
- `denied to OLD_ACCOUNT`: GitHub가 대상 저장소 권한이 없는 계정으로 인증했다.

## 원인

이 문제는 보통 아래 조건이 겹칠 때 발생한다.

1. 저장소에 이미 `origin` remote가 등록되어 있다.
2. GitHub 계정을 여러 개 쓰고 있다.
3. 기본 `github.com` SSH key나 HTTPS credential이 `OLD_ACCOUNT`로 인증된다.
4. 하지만 대상 저장소 `TARGET_OWNER/REPO`에 push 권한이 있는 계정은 `TARGET_ACCOUNT`다.

즉, Git 설정의 `user.name`과 `user.email`은 커밋 작성자 정보일 뿐, GitHub push 인증 계정을 바꾸지 않는다.

## 빠른 해결

대상 계정 전용 SSH key를 만든다.

```bash
ssh-keygen -t ed25519 -C "EMAIL@example.com" -f ~/.ssh/id_ed25519_target
```

공개키를 출력한다.

```bash
cat ~/.ssh/id_ed25519_target.pub
```

출력된 공개키를 GitHub의 `TARGET_ACCOUNT` 계정에 등록한다. GitHub에서는 `Settings` → `SSH and GPG keys` → `New SSH key`에서 등록한다.

`~/.ssh/config`에 대상 계정용 Host alias를 추가한다.

```bash
cat >> ~/.ssh/config <<'EOF'
Host github-target
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_target
  IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/config
```

저장소 remote를 기본 `github.com`이 아니라 방금 만든 alias로 바꾼다.

```bash
git remote set-url origin git@github-target:TARGET_OWNER/REPO.git
```

SSH 인증 계정을 확인하고 push한다.

```bash
ssh -T git@github-target
git push -u origin main
```

## 왜 Host alias가 필요한가

SSH remote는 보통 다음처럼 쓴다.

```bash
git@github.com:OWNER/REPO.git
```

하지만 GitHub 계정을 여러 개 쓰면 `github.com` 하나만으로는 어떤 SSH key를 써야 하는지 구분하기 어렵다. 이때 `~/.ssh/config`에서 `github-target` 같은 alias를 만들면, Git remote마다 사용할 SSH key를 명시할 수 있다.

```text
Host github-target
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_target
  IdentitiesOnly yes
```

그다음 remote는 실제 GitHub 호스트가 아니라 alias를 사용한다.

```bash
git@github-target:TARGET_OWNER/REPO.git
```

이렇게 하면 해당 저장소는 항상 `~/.ssh/id_ed25519_target` key로 인증한다.

## 확인 체크리스트

```bash
git remote -v
git branch --show-current
git status
ssh -T git@github-target
```

확인할 점은 다음과 같다.

- `origin`이 `git@github-target:TARGET_OWNER/REPO.git` 형식인가?
- `ssh -T git@github-target`가 대상 GitHub 계정으로 인증되는가?
- 공개키가 대상 GitHub 계정에 등록되어 있는가?
- 현재 branch가 push하려는 branch인가?

## 재발 방지

새 저장소를 연결할 때는 `git remote add origin ...`을 바로 실행하기 전에 항상 remote를 먼저 확인한다.

```bash
git remote -v
```

이미 `origin`이 있으면 `add`가 아니라 `set-url`을 쓴다.

```bash
git remote set-url origin git@github-target:TARGET_OWNER/REPO.git
```

커밋 작성자 정보와 GitHub 인증 계정은 별개라는 점도 기억해야 한다.

```bash
git config user.name "DISPLAY_NAME"
git config user.email "EMAIL@example.com"
```

위 설정은 커밋 메타데이터만 바꾼다. push 권한 문제는 remote URL과 GitHub 인증 계정에서 해결해야 한다.

## Related

- [[git/_index|Git]]
