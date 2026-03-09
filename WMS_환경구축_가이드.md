# 🧬 생물정보학 분석 환경 구축 가이드
> MetaPhlAn (WMS 분석) 설치를 기반으로 작성된 가이드입니다.  
> 다른 패키지 설치 시에도 동일한 흐름으로 활용할 수 있습니다.

---

## 📋 목차
1. [사전 확인](#1-사전-확인)
2. [가상환경 생성](#2-가상환경-생성)
3. [패키지 설치 우선순위](#3-패키지-설치-우선순위)
4. [충돌 에러 해결 패턴](#4-충돌-에러-해결-패턴)
5. [공용 모듈 사용법](#5-공용-모듈-사용법)
6. [대용량 작업 시 tmux 필수](#6-대용량-작업-시-tmux-필수)
7. [MetaPhlAn 설치 전체 흐름](#7-metaphlan-설치-전체-흐름)
8. [다른 패키지 설치 시 체크리스트](#8-다른-패키지-설치-시-체크리스트)

---

## 1. 사전 확인

```bash
# conda 설치 확인
conda --version

# 공용 모듈 확인
module avail
```

> 💡 **핵심 개념**
> - `module avail` : 서버에서 공용으로 제공하는 툴 목록
> - `conda` : 개인 가상환경 관리자로 패키지 충돌을 방지해줌
> - conda가 없을 경우 [Miniconda 공식 설치 페이지](https://docs.conda.io/en/latest/miniconda.html) 참고

---

## 2. 가상환경 생성

```bash
# 환경 생성 (Python 버전 명시가 핵심!)
conda create -n 환경이름 python=3.8 -y

# 활성화
conda activate 환경이름

# 비활성화
conda deactivate

# 환경 목록 확인
conda env list

# 환경 삭제 (잘못 만들었을 때)
conda env remove -n 환경이름
```

> 💡 **핵심 개념**
> - Python 버전을 **명시하지 않으면** 시스템 최신 버전(3.13 등)이 설치되어 패키지 충돌 발생
> - 패키지마다 **권장 Python 버전이 다름** → 설치 전 공식 문서 확인 필수
> - 가상환경은 **서로 독립적** → 한 환경이 망가져도 다른 환경에 영향 없음

### source vs conda activate 차이점

| 구분 | `source` 방식 | `conda activate` 방식 |
|------|--------------|----------------------|
| 환경 종류 | Python venv / virtualenv | Conda 환경 |
| 활성화 명령 | `source venv/bin/activate` | `conda activate 환경명` |
| Python 버전 관리 | ❌ 불가 | ✅ 환경마다 지정 가능 |
| non-Python 패키지 | ❌ Python만 | ✅ bowtie2 등 바이너리도 가능 |
| 생물정보학 적합성 | ⚠️ 제한적 | ✅ 권장 |

---

## 3. 패키지 설치 우선순위

```bash
# 1순위 - conda (bioconda: 생물정보학 패키지 전용 채널)
conda install -c bioconda -c conda-forge 패키지명 -y

# 2순위 - pip (conda 실패 시 차선책)
pip install 패키지명

# 설치 확인
패키지명 --version

# 현재 환경에 설치된 패키지 목록
conda list
```

> 💡 **핵심 개념**
> - **conda 우선 권장** : Python 외 의존성(bowtie2 등)도 자동 해결
> - **pip 차선책** : Python 패키지만 처리, 나머지 의존성은 수동 설치 필요
> - `-c bioconda` : 생물정보학 전용 채널 (MetaPhlAn, QIIME 등 대부분 여기에 있음)
> - `-c conda-forge` : 범용 커뮤니티 채널, 최신 패키지 보유

### 채널 확인 및 추가

```bash
# 현재 채널 확인
conda config --show channels

# 채널 추가
conda config --add channels defaults
conda config --add channels conda-forge
conda config --add channels bioconda
```

---

## 4. 충돌 에러 해결 패턴

### 에러 메시지 예시
```
LibMambaUnsatisfiableError: Encountered problems while solving
- pin on python=3.9 * is not installable
- urllib3 conflicts with any installable versions previously reported
```

### 해결 순서

```
LibMambaUnsatisfiableError 발생
        ↓
에러 메시지 하단 확인
"pin on python=X.X" → Python 버전 충돌
"urllib3/requests conflict" → 의존성 충돌
        ↓
해결 방법 순서:
1. Python 버전 낮춰서 환경 재생성 (3.8 권장)
2. conda 대신 pip으로 설치
3. mamba 사용 (conda보다 의존성 해결 능력 우수)
```

```bash
# 방법 1 - Python 버전 낮춰서 재생성
conda env remove -n 환경이름
conda create -n 환경이름 python=3.8 -y
conda activate 환경이름
conda install -c bioconda -c conda-forge 패키지명 -y

# 방법 2 - pip으로 설치
pip install 패키지명

# 방법 3 - mamba 사용
conda install -c conda-forge mamba -y
mamba install -c bioconda -c conda-forge 패키지명 -y
```

### Python 버전 호환성 참고

| Python 버전 | 일반적 호환성 |
|------------|-------------|
| 3.13 | ❌ 너무 최신, 대부분 패키지 미지원 |
| 3.11 / 3.12 | ⚠️ 불안정할 수 있음 |
| **3.8 / 3.9** | ✅ 생물정보학 패키지 권장 버전 |
| 3.7 | ✅ 가능하나 오래됨 |

---

## 5. 공용 모듈 사용법

```bash
# 모듈 목록 확인
module avail

# 모듈 로드 (버전 번호 반드시 포함!)
module add 툴이름/버전번호

# 예시
module add bowtie2/2.5.4
module add samtools/1.22.1

# 로드 확인
bowtie2 --version

# 현재 로드된 모듈 확인
module list

# 모듈 해제
module rm 툴이름/버전번호

# 매번 로드하기 귀찮으면 ~/.bashrc에 등록 (영구 설정)
echo 'module add bowtie2/2.5.4' >> ~/.bashrc
source ~/.bashrc
```

> ⚠️ **주의사항**
> - 버전 번호 없이 `module add bowtie2` 입력 시 에러 발생
> - conda 환경 활성화 상태에서 module이 충돌할 수 있음
> - 충돌 시 → conda 환경 안에 직접 설치하는 것이 더 안전

---

## 6. 대용량 작업 시 tmux 필수

### tmux가 필요한 이유
```
tmux 없이 실행 시:
PC 종료 or VSCode 종료 → SSH 끊김 → 작업 중단 ❌

tmux 사용 시:
PC 종료 or VSCode 종료 → SSH 끊김 → tmux는 서버에서 계속 실행 ✅
```

### tmux 기본 사용법

```bash
# 새 세션 시작
tmux new -s 세션이름

# 백그라운드로 전환 (detach)
# → Ctrl + B 누르고 손 뗀 후 → D 누르기
Ctrl + B → D

# 세션 목록 확인
tmux ls

# 세션 재접속
tmux attach -t 세션이름

# 세션 종료
exit
```

### tmux 주요 단축키

| 동작 | 단축키 |
|------|--------|
| 백그라운드 전환 (detach) | `Ctrl+B` → `D` |
| 새 창 만들기 | `Ctrl+B` → `C` |
| 창 전환 | `Ctrl+B` → `N` |
| 세션 목록 | `tmux ls` |
| 세션 재접속 | `tmux attach -t 세션이름` |

> 💡 **단축키 사용법**
> - `Ctrl+B` 를 **동시에** 누른 후 **손을 떼고**
> - 그 다음 키(`D`, `C`, `N` 등)를 **혼자** 누르기
> - 타이핑 후 엔터가 아님!

> ⚠️ **주의사항**
> - 서버 재부팅 시에만 세션이 종료됨
> - VSCode 종료, PC 종료는 영향 없음

---

## 7. MetaPhlAn 설치 전체 흐름

### 설치 흐름 요약

```
1. conda 가상환경 생성 (python=3.8)
        ↓
2. pip으로 metaphlan 설치 (conda 충돌로 pip 사용)
        ↓
3. conda로 bowtie2 설치 (의존성 툴)
        ↓
4. metaphlan_db 폴더 생성
        ↓
5. tmux 세션 시작
        ↓
6. DB 다운로드 (약 33GB, 2시간 소요)
        ↓
7. Ctrl+B → D 로 백그라운드 전환
```

### 실제 명령어

```bash
# 1. 가상환경 생성
conda create -n metaphlan_env python=3.8 -y
conda activate metaphlan_env

# 2. MetaPhlAn 설치
pip install metaphlan

# 3. bowtie2 설치
conda install -c bioconda bowtie2 -y

# 4. DB 폴더 생성
mkdir -p ~/metaphlan_db

# 5. tmux 세션 시작
tmux new -s metaphlan_db

# 6. DB 다운로드 (신버전 옵션: --db_dir)
conda activate metaphlan_env
metaphlan --install --db_dir ~/metaphlan_db

# 7. 백그라운드 전환
# Ctrl + B → D

# 8. 나중에 완료 확인
tmux attach -t metaphlan_db
ls -lh ~/metaphlan_db/
```

### 분석 실행

```bash
conda activate metaphlan_env

metaphlan input.fastq.gz \
  --db_dir ~/metaphlan_db \
  --input_type fastq \
  -o output_profile.txt \
  --nproc 8
```

---

## 8. 다른 패키지 설치 시 체크리스트

```
✅ 공식 문서에서 권장 Python 버전 확인
✅ conda create 시 Python 버전 명시
✅ conda install 먼저 시도
✅ 실패 시 에러 메시지 하단 확인
✅ conda 실패 시 pip으로 대체
✅ 의존성 툴 별도 확인 (bowtie2, samtools 등)
✅ 대용량 설치는 tmux 세션에서 실행
✅ 설치 완료 후 --version으로 확인
```

---

## 📁 권장 디렉토리 구조

```
/home/username/
├── miniconda3/              # conda 설치 경로
├── metaphlan_db/            # MetaPhlAn DB (레퍼런스)
└── analysis/
    ├── rawdata/             # 원본 데이터
    ├── results/             # 분석 결과
    └── scripts/             # 분석 스크립트
```

---

## 🔗 참고 링크

- [MetaPhlAn 공식 문서](https://github.com/biobakery/MetaPhlAn)
- [Bioconda 패키지 검색](https://bioconda.github.io/)
- [Conda 공식 문서](https://docs.conda.io/)
- [tmux 치트시트](https://tmuxcheatsheet.com/)

---

> 📝 **작성 기록**  
> 작성일: 2026-03-09  
> 분석 환경: Linux (Ubuntu), Miniconda3  
> 분석 목적: WMS (Whole Metagenome Shotgun) 분석
