# teamprex/asyncmy fork

prex-core(`python/pyproject.toml`)가 PyPI `asyncmy` 대신 이 fork를 쓰는 이유와, fork를 없앨 수 있는 조건을 적어요.
fork를 바꾸거나 없앨 때 이 파일도 같이 고쳐 주세요.

## 현재: `prex/v0.2.16` 브랜치 (2026-11-05)

upstream `v0.2.16` 태그에 아래 패치 하나만 얹었어요. 그 외 코드는 upstream과 같아요.

### 패치: `_MySQLProtocol.connection_lost`가 예외를 traceback 없이 저장

- **문제**
  - 0.2.13부터 연결 계층이 asyncio StreamReader에서 자체 `_MySQLProtocol`로 바뀌었어요.
  - 새 프로토콜은 `connection_lost(exc)`의 예외를 `self.exc = exc`로 그대로 들고 있어요.
  - 이 예외의 traceback → asyncio SSL transport → 프로토콜로 순환 참조가 생겨요.
  - 그래서 끊긴 TLS 연결 하나마다 다음이 full GC 때까지 남아요.
    - 프로토콜과 읽기 버퍼
    - SSL transport의 256 KiB 버퍼와 OpenSSL 상태
  - 0.2.10~0.2.12의 StreamReader 경로는 `connection_lost`에서 참조를 끊어서 이 문제가 없었어요.
- **영향 (prex)**
  - prex-core는 `pool_recycle=180`, `pool_pre_ping=True`, MySQL TLS를 써서 끊긴 TLS 연결이 계속 생겨요.
  - 할당이 적어 GC가 드물게 도는 한가한 서비스에서 메모리가 계속 늘었어요.
    - prod `be-user-activity-dispatcher`: 13시간에 428MB → 1.2GB (limit 1.6GB)
    - stg: 20분 주기로 +70MB 톱니
  - 바쁜 서비스는 GC가 자주 돌아서 드러나지 않았어요.
- **확인 방법 (stg, 같은 `release-3.45.4` 이미지에서 asyncmy만 교체)**

  | asyncmy | dispatcher 메모리 |
  |---|---|
  | 0.2.16 (PyPI) | 분당 약 4MB 증가, 20분마다 full GC로 떨어짐 |
  | 0.2.10 (이전 fork) | 평탄 |
  | 0.2.16 + 이 패치 | 평탄 |

  tracemalloc과 gc 타입 집계에서 `SSLError`, `traceback`, `_MySQLProtocol`, `SSLContext`가 같이 늘었고, `asyncio/sslproto.py` 256 KiB 버퍼 할당이 잡혔어요.
- **upstream**: https://github.com/long2ice/asyncmy/pull/171 (2026-11-05). 머지되고 릴리스되면 아래 조건대로 fork를 없애요.
- **prex-core**: Prex/prex-core#15037 (fork 고정), release-3.45.24부터 포함.

### 이 fork를 없앨 수 있는 조건

- upstream 릴리스에 같은 수정(또는 `connection_lost` 이후 참조를 끊는 수정)이 들어가면, prex-core를 그 PyPI 버전으로 바꾸고 이 fork 의존성을 지워요.
- 바꾼 뒤에는 stg에서 한가한 서비스(`be-user-activity-dispatcher`)의 메모리를 몇 시간 이상 확인해 주세요. 기능 테스트로는 안 잡혀요.

### 0.2.12로 내리면 안 되나

- CVE-2025-65896 / GHSA-qhqw-rrw9-25rm (dict 키 SQL injection, CVSS 9.8)은 0.2.11까지 영향이 있어요. 그래서 최소 0.2.12가 필요해요.
- 0.2.12는 이 누수가 없는 StreamReader 경로예요. 다만 다음 두 가지 때문에 0.2.16 + 패치를 골랐어요.
  - PyPI에 linux aarch64 cp312 wheel이 없어요.
  - 0.2.13~0.2.16의 버그 수정을 잃어요. 예: `pool_pre_ping`이 닫힌 transport를 인식하는 수정.

## 이전 이력

- **`prex-dev` 브랜치 (`6eedc98`, 2025-05)**: upstream 0.2.10 + Cython `<3.1.0` 고정.
  - Cython 3.1.0의 메모리 누수(cython#6850, #6878)를 피하려고 만들었어요. 3.1.1에서 고쳐졌어요.
- **2026-10 prex-core #14932**: CVE 때문에 PyPI 0.2.16으로 옮기면서 fork를 쓰지 않게 됐어요.
- **2026-11 이 브랜치**: 위 누수 때문에 fork로 다시 돌아왔어요.
