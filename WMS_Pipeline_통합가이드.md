# 🦠 Gut Microbiome Analysis Pipeline & 환경 구축 가이드

> MetaPhlAn4 & StrainPhlAn 기반 WMS 분석 파이프라인 및 환경 구축 통합 가이드
> 작성: Kyujin Paik | 최초: 2026-03-09 | 업데이트: 2026-03-24

---

## 📋 목차

1. [분석 개요](#1-분석-개요)
2. [프로젝트 구조](#2-프로젝트-구조)
3. [샘플 정보](#3-샘플-정보)
4. [환경 구축](#4-환경-구축)
5. [분석 파이프라인](#5-분석-파이프라인)
6. [Downstream 분석 R](#6-downstream-분석-r)
7. [핵심 개념 정리](#7-핵심-개념-정리)
8. [트러블슈팅](#8-트러블슈팅)
9. [참고 링크](#9-참고-링크)

---

## 1. 분석 개요

```
Raw FASTQ (paired-end)
        │
        ▼
┌─────────────────────────────────────────────────────┐
│                    MetaPhlAn4                        │
│  --mapout  →  .bowtie2.bz2  (재분석용 요약본)         │
│  --samout  →  .sam.bz2      (StrainPhlAn용 전체본)   │
│  -o        →  profile.txt   (종분포도)               │
└─────────────────────────────────────────────────────┘
        │
        ├──────────────────────────────────────────────┐
        ▼                                              ▼
┌───────────────┐                          ┌──────────────────┐
│  종분포도 분석  │                          │   StrainPhlAn    │
│  (MetaPhlAn4) │                          │  (균주 계통분석)  │
└───────────────┘                          └──────────────────┘
        │                                              │
        ▼                                              ▼
Relative Abundance Table              consensus markers (.json.bz2)
Read Count Table                      계통수 (phylogenetic tree)
        │
        ▼
PCoA / MaAsLin2 / ANCOM-BC / DESeq2
```

---

## 2. 프로젝트 구조

```
/home/paik/
├── miniconda3/
└── 2.ASM/
    ├── rawdata/                        # Raw paired-end FASTQ (65 samples)
    │   ├── ASM_01_001_1.fastq.gz
    │   ├── ASM_01_001_2.fastq.gz
    │   └── ...
    └── metaphlan4_out/
        ├── bowtie2/                    # .bowtie2.bz2 (mapout) ← MetaPhlAn 재분석용
        ├── sam/                        # .sam.bz2     (samout) ← StrainPhlAn용
        ├── profiles/                   # 종분포도 (relative abundance)
        ├── profiles_counts/            # 카운트 테이블 (ANCOM-BC / DESeq2용)
        ├── merged/
        │   ├── merged_abundance_table.txt
        │   └── merged_counts_table.txt
        └── strainphlan/
            └── consensus_markers/      # 샘플별 균주 대표서열 (.json.bz2)
```

---

## 3. 샘플 정보

| Group | Sample Prefix | N |
|-------|--------------|---|
| Group 1 | ASM_01 | 15 |
| Group 2 | ASM_02 | 16 |
| Group 3 | ASM_03 | 24 |
| Group 4 | KNU_03 | 10 |
| **Total** | | **65** |

---

## 4. 환경 구축

### 4-1. 사전 확인

```bash
conda --version
module avail
```

### 4-2. 가상환경 생성

```bash
# Python 버전 명시 필수!
conda create -n metaphlan_env python=3.8 -y
conda activate metaphlan_env
conda deactivate

conda env list
conda env remove -n 환경이름   # 잘못 만들었을 때
```

> 💡 Python 버전을 명시하지 않으면 최신 버전(3.13 등)이 설치되어 패키지 충돌 발생

| Python 버전 | 호환성 |
|------------|--------|
| 3.13 | ❌ 대부분 패키지 미지원 |
| 3.11 / 3.12 | ⚠️ 불안정 |
| **3.8 / 3.9** | ✅ 생물정보학 권장 |

### 4-3. 패키지 설치 우선순위

```bash
# 1순위 - conda (bioconda: 생물정보학 전용 채널)
conda install -c bioconda -c conda-forge 패키지명 -y

# 2순위 - pip (conda 실패 시)
pip install 패키지명

# 3순위 - mamba (의존성 해결 능력 우수)
conda install -c conda-forge mamba -y
mamba install -c bioconda -c conda-forge 패키지명 -y
```

### 4-4. 공용 모듈 사용법

```bash
module avail                        # 모듈 목록 확인
module add bowtie2/2.5.4            # 버전 번호 반드시 포함!
module add samtools/1.22.1
module list                         # 현재 로드된 모듈 확인
module rm bowtie2/2.5.4             # 모듈 해제

# 영구 등록 (~/.bashrc)
echo 'module add bowtie2/2.5.4' >> ~/.bashrc
source ~/.bashrc
```

### 4-5. tmux 필수 사용법

```
tmux 없이: PC/VSCode 종료 → SSH 끊김 → 작업 중단 ❌
tmux 사용: PC/VSCode 종료 → SSH 끊김 → 서버에서 계속 실행 ✅
```

```bash
tmux new -s 세션이름        # 새 세션 시작
tmux ls                    # 세션 목록
tmux attach -t 세션이름     # 세션 재접속
exit                       # 세션 종료
```

| 동작 | 단축키 |
|------|--------|
| 백그라운드 전환 (detach) | `Ctrl+B` → `D` |
| 새 창 만들기 | `Ctrl+B` → `C` |
| 창 전환 (번호로) | `Ctrl+B` → `0`, `1`, `2`... |
| 세션 목록 보기 | `Ctrl+B` → `S` |

> 💡 `Ctrl+B` 동시에 누른 후 **손을 떼고** 다음 키를 누를 것

### 4-6. MetaPhlAn4 설치 전체 흐름

```bash
# 1. 가상환경 생성
conda create -n metaphlan_env python=3.8 -y
conda activate metaphlan_env

# 2. MetaPhlAn 설치
pip install metaphlan

# 3. bowtie2 설치
conda install -c bioconda bowtie2 -y

# 4. DB 다운로드 (약 33GB, 2시간 소요 → tmux 필수!)
tmux new -s metaphlan_db
conda activate metaphlan_env
metaphlan --install --db_dir /opt/refs/bacteria/metaphlan_db
# Ctrl+B → D 로 백그라운드 전환
```

---

## 5. 분석 파이프라인

### STEP 1: MetaPhlAn4 실행

> ⚠️ **핵심**: StrainPhlAn을 계획하고 있다면 `--samout`을 **처음부터 반드시** 포함!
> `--mapout`과 `--samout`은 동일한 매핑을 다른 형식으로 저장하므로 성능 손실 없음.

```bash
mkdir -p $HOME/2.ASM/metaphlan4_out/sam

for f in $HOME/2.ASM/rawdata/*_1.fastq.gz; do
    SAMPLE=$(basename "$f" _1.fastq.gz)
    metaphlan \
        "$HOME/2.ASM/rawdata/${SAMPLE}_1.fastq.gz,$HOME/2.ASM/rawdata/${SAMPLE}_2.fastq.gz" \
        --input_type fastq \
        --db_dir /opt/refs/bacteria/metaphlan_db \
        --mapout $HOME/2.ASM/metaphlan4_out/bowtie2/${SAMPLE}.bowtie2.bz2 \
        --samout $HOME/2.ASM/metaphlan4_out/sam/${SAMPLE}.sam.bz2 \
        --nproc 5 \
        --tax_lev a \
        -o $HOME/2.ASM/metaphlan4_out/profiles/${SAMPLE}_profile.txt
done
```

| 출력 파일 | 용도 |
|----------|------|
| `.bowtie2.bz2` | MetaPhlAn 파라미터 변경 후 재분석 시 매핑 스킵 |
| `.sam.bz2` | StrainPhlAn 입력 |
| `_profile.txt` | PCoA, MaAsLin2 |

### STEP 2: Read Count Table 생성 (ANCOM-BC / DESeq2용)

```bash
for f in $HOME/2.ASM/metaphlan4_out/bowtie2/*.bowtie2.bz2; do
    SAMPLE=$(basename "$f" .bowtie2.bz2)
    metaphlan \
        "$f" \
        --input_type mapout \
        -t rel_ab_w_read_stats \
        --db_dir /opt/refs/bacteria/metaphlan_db \
        -o $HOME/2.ASM/metaphlan4_out/profiles_counts/${SAMPLE}_counts.txt
done
```

### STEP 3: 테이블 병합

```bash
# Relative abundance table
merge_metaphlan_tables.py \
    $HOME/2.ASM/metaphlan4_out/profiles/*_profile.txt \
    -o $HOME/2.ASM/metaphlan4_out/merged/merged_abundance_table.txt

# Read count table
merge_metaphlan_tables.py \
    $HOME/2.ASM/metaphlan4_out/profiles_counts/*_counts.txt \
    -o $HOME/2.ASM/metaphlan4_out/merged/merged_counts_table.txt
기본 merge_metaphlan_tables.py 스크립트는 relative_abundance 컬럼을 우선적으로 병합하도록 하드코딩되어 있어, ANCOM-BC2 분석에 필요한 estimated count(추정 읽기 수) 대신 퍼센트(%) 비율이 결과로 나오게 됩니다. 
GitHub
GitHub
 +1
기억하시는 대로, GitHub에서 수정된 merge_metaphlan4_tables_abs.py 스크립트를 사용해야 estimated_number_of_reads_from_the_clade 컬럼을 정확히 병합할 수 있습니다. 
GitHub
GitHub
 +1
수정된 스크립트 다운로드 및 사용법
아래 명령어를 통해 timyerg님이 공개한 MetaPhlAn 4 전용 수정 스크립트를 다운로드하여 실행하세요.
```

### STEP 4: StrainPhlAn - consensus markers 생성

> ⚠️ `-f bz2`는 MetaPhlAn3 전용. MetaPhlAn4의 `.sam.bz2`는 인식 불가.
> 반드시 `bzcat`으로 압축 해제 후 `-f sam`으로 입력.

```bash
mkdir -p $HOME/2.ASM/metaphlan4_out/strainphlan/consensus_markers

for f in $HOME/2.ASM/metaphlan4_out/sam/*.sam.bz2; do
    SAMPLE=$(basename "$f" .sam.bz2)

    # 압축 해제 → 임시 SAM 생성
    bzcat "$f" > /tmp/${SAMPLE}.sam

    sample2markers.py \
        -i /tmp/${SAMPLE}.sam \
        -o $HOME/2.ASM/metaphlan4_out/strainphlan/consensus_markers \
        -n 5 \
        -d /opt/refs/bacteria/metaphlan_db/mpa_vJan25_CHOCOPhlAnSGB_202503.pkl \
        -f sam

    rm /tmp/${SAMPLE}.sam   # 임시 파일 삭제
done
```

출력: `consensus_markers/SAMPLE.json.bz2` (샘플별 균주 대표서열)

### STEP 5: StrainPhlAn - 계통분석

```bash
# 분석 가능한 clade 확인
strainphlan.py \
    --samples $HOME/2.ASM/metaphlan4_out/strainphlan/consensus_markers/*.json.bz2 \
    --database /opt/refs/bacteria/metaphlan_db/mpa_vJan25_CHOCOPhlAnSGB_202503.pkl \
    --print_clades_only \
    -o $HOME/2.ASM/metaphlan4_out/strainphlan/

# 특정 clade 계통수 생성
strainphlan.py \
    --samples $HOME/2.ASM/metaphlan4_out/strainphlan/consensus_markers/*.json.bz2 \
    --database /opt/refs/bacteria/metaphlan_db/mpa_vJan25_CHOCOPhlAnSGB_202503.pkl \
    --clade t__SGB_XXXXX \
    -o $HOME/2.ASM/metaphlan4_out/strainphlan/ \
    --nprocs 5
```

---

## 6. Downstream 분석 (R)

| 분석 | 입력 | 목적 |
|------|------|------|
| **PCoA** (vegan) | Relative abundance | Beta diversity 시각화 |
| **MaAsLin2** | Relative abundance | 다변량 연관성 분석 |
| **ANCOM-BC** | Read counts (integer) | 차등 풍부도 분석 (bias 보정) |
| **DESeq2** | Read counts (integer) | 차등 풍부도 분석 |
| **StrainPhlAn** | consensus markers | 균주 수준 계통 분석 |

---

## 7. 핵심 개념 정리

### --mapout vs --samout

```
동일한 Bowtie2 매핑 (1번만 실행)
              ↓
        매핑 결과 원본
       ↙              ↘
  --mapout           --samout
  요약본 저장         전체 저장 (표준 SAM)

  MetaPhlAn만 읽음    StrainPhlAn 등 외부 툴 활용 가능
  종분포도 재분석용    역변환 불가 ❌ → 완전한 정보 보존 ✅
```

| | `--mapout` | `--samout` |
|---|---|---|
| **저장 내용** | MetaPhlAn 필요 정보만 (요약본) | 매핑 결과 전체 (표준 SAM) |
| **역변환** | ❌ SAM 복원 불가 (손실 포맷) | ✅ 완전한 정보 보존 |
| **용도** | MetaPhlAn 재분석 시 매핑 스킵 | StrainPhlAn 등 외부 툴 입력 |
| **StrainPhlAn 호환** | ❌ | ✅ (압축 해제 후) |

> 💡 **버전 차이**: MetaPhlAn3의 `--bowtie2out` 출력물은 StrainPhlAn에서 `-f bz2`로 직접 사용 가능했으나,
> MetaPhlAn4의 `--mapout`은 내부 포맷이 변경되어 **호환 불가**. 반드시 `--samout` 별도 지정 필요.

### SAM vs BAM

| | SAM | BAM |
|---|---|---|
| **형식** | 텍스트 | 바이너리 압축 |
| **인덱싱** | ❌ 불가 | ✅ 가능 |
| **처리** | 처음부터 순서대로 읽어야 함 | 특정 위치로 바로 점프 가능 |

`sample2markers.py`가 SAM → BAM 변환하는 이유: 수백 개 marker gene 위치로 **바로 점프**하여 pileup 수행하기 위함

---

## 8. 트러블슈팅

| Error | 원인 | 해결 |
|-------|------|------|
| `unrecognized arguments: --bowtie2out` | 구버전 파라미터 | `--mapout` 사용 |
| `unrecognized arguments: --bowtie2db` | 구버전 파라미터 | `--db_dir` 사용 |
| `invalid choice: bowtie2out` | 구버전 파라미터 | `--input_type mapout` 사용 |
| `Input file not found: ~/path` | `~` 경로 미확장 | `$HOME` 사용 |
| `The input format must be SAM, BAM, or compressed in BZ2 format` | `.sam.bz2` 직접 입력 | `bzcat`으로 압축 해제 후 `-f sam` 입력 |
| `ValueError: parsing SAM record string failed` | mapout 파일을 sample2markers에 입력 | `--samout`으로 재실행 필요 (역변환 불가) |
| `Mapping output file detected` | 기존 `.bowtie2.bz2` 존재 | 기존 파일 삭제 후 재실행 |

### 설치 시 충돌 해결 순서

```
LibMambaUnsatisfiableError 발생
        ↓
1. Python 버전 낮춰서 환경 재생성 (python=3.8)
2. conda 대신 pip 사용
3. mamba 사용
```

### 패키지 설치 체크리스트

```
✅ 공식 문서에서 권장 Python 버전 확인
✅ conda create 시 Python 버전 명시
✅ conda install 먼저 시도 → 실패 시 pip
✅ 의존성 툴 별도 확인 (bowtie2, samtools 등)
✅ 대용량 설치는 tmux 세션에서 실행
✅ 설치 완료 후 --version으로 확인
✅ StrainPhlAn 계획 시 --samout 처음부터 포함 여부 확인
```

---

## 9. 참고 링크

- [MetaPhlAn4 Documentation](https://github.com/biobakery/MetaPhlAn)
- [StrainPhlAn Tutorial](https://github.com/biobakery/MetaPhlAn/wiki/StrainPhlAn-4)
- [MaAsLin2](https://github.com/biobakery/Maaslin2)
- [ANCOM-BC](https://github.com/FrederickHuangLin/ANCOMBC)
- [DESeq2](https://bioconductor.org/packages/release/bioc/html/DESeq2.html)
- [Bioconda 패키지 검색](https://bioconda.github.io/)
- [tmux 치트시트](https://tmuxcheatsheet.com/)
