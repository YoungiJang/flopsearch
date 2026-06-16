# FLOP 실행 가이드 (run_guide)

이 저장소의 FLOP 인과 발견 알고리즘을 **Rust CLI** 또는 **Python**으로 실행하는 방법.
(R은 사용하지 않음)

## 환경 구성 요약 (이미 설치 완료)

| 구성 요소 | 위치 | 비고 |
|-----------|------|------|
| Rust 툴체인 1.96.0 | `~/.cargo`, `~/.rustup` | 사용자 로컬, 전역 영향 없음 |
| Rust CLI 바이너리 | `flop/target/release/flop` | `cargo build --release` 산출물 |
| Python 환경 (miniforge conda, py3.12) | `.env/` | 프로젝트 내 격리. base는 conda-forge, maturin·numpy·scipy는 PyPI |
| Python 모듈 `flopsearch` | `.env`에 editable 설치 | `maturin develop` 산출물 |

> Python 환경을 conda로 만든 이유: 시스템 파이썬이 externally-managed(PEP 668)이고 `python3-venv`가 없어(설치에 sudo 필요), 평소 쓰는 **miniforge**(`~/miniforge3`, conda-forge 채널)로 만듦.

---

## 1. Rust CLI 로 실행

추가 의존성 없음. 새 터미널에서는 PATH 등록만 한 번 필요.

```bash
source "$HOME/.cargo/env"          # 새 터미널에서만 1회
cd /mnt/nfs/colossal/youngin/flopsearch/flop
./target/release/flop <data.csv> 2.0 --restarts 50
```

- `<data.csv>`: 첫 줄이 변수명 헤더, 이후 각 행이 관측치인 데이터 행렬
- `2.0`: BIC 페널티 파라미터 (권장 기본값)
- `--restarts 50`: ILS 재시작 횟수. 대신 `--timeout <초>` 로 시간 제한 가능

출력: `<노드수> <엣지수> cpdag` 헤더 뒤에 엣지 목록(`i j directed` 또는 `i j undirected`).

---

## 2. Python 으로 실행

```bash
source /mnt/nfs/colossal/youngin/miniforge3/etc/profile.d/conda.sh
conda activate /mnt/nfs/colossal/youngin/flopsearch/.env
```

```python
import flopsearch
import numpy as np
from scipy import linalg

p = 10
W = np.diag(np.ones(p - 1), 1)
X = np.random.randn(10000, p).dot(linalg.inv(np.eye(p) - W))
X_std = (X - np.mean(X, axis=0)) / np.std(X, axis=0)

cpdag = flopsearch.flop(X_std, 2.0, restarts=50)   # numpy 행렬 반환
```

- 입력 `X_std`: (관측치 × 변수) numpy 행렬
- 인자는 CLI와 동일: BIC 페널티, `restarts` 또는 `timeout`
- 출력: CPDAG 인접행렬 (numpy). 항목 `1` = i→j 방향 엣지, `2` = i—j 무방향 엣지
  (무방향 엣지는 (i,j), (j,i) 양쪽에 `2`)

---

## 3. 코드 수정 후 재빌드

| 대상 | 명령 |
|------|------|
| Rust 코어 / CLI | `cd flop && cargo build --release` |
| Python 모듈 | `conda activate .env` 후 `cd flop_python && maturin develop --release` |

(둘 다 `flop/` 코어를 공유하므로, 코어를 고치면 사용하는 쪽을 다시 빌드)
