# OME-TIFF & SVS 생체 이미지 포맷 분석 및 OpenSlide 실습 가이드

본 문서는 현미경 및 생체 이미지 분석용 핵심 공용 포맷 개요와 함께 **SVS (Aperio Digital Pathology) 포맷의 메타데이터 추출, 영역 타일링, 그리고 OpenSlide / DeepZoom을 활용한 대용량 멀티프로세싱 변환 파이썬 실습 코드**를 종합 정리한 가이드입니다.

---

## 1. 현미경 및 생체 이미지 분석용 핵심 공용 포맷 개요

### 1.1 핵심 공용 포맷
1. **OME-TIFF (`.ome.tif` / `.ome.tiff`)**
   - **설명:** 오픈 현미경 협회(Open Microscopy Environment)에서 개발한 생물학 이미지 표준 공용 포맷.
   - **특징:** 현미경 배율, 레이저 파장, Z-stack 간격, 채널 정보 등 핵심 메타데이터를 XML 형태로 완전 보존하며 무손실 압축(TIFF)으로 저장. LIF 등 독자 포맷 변환 시 가장 권장됨.
2. **TIFF / Multi-page TIFF (`.tif` / `.tiff`)**
   - **설명:** 가장 널리 쓰이는 표준 무손실 그래픽 포맷.
   - **특징:** 시계열(Time-lapse)이나 Z-stack 3차원 생체 이미지 묶음을 하나의 Multi-page TIFF 파일로 내보낼 수 있음. 단, 현미경 특유의 세부 메타데이터가 일부 유실될 수 있음.
3. **OME-Zarr (`.zarr`)**
   - **설명:** 차세대 생체 이미지 표준으로 급부상하고 있는 클라우드 최적화 포맷.
   - **특징:** 테라바이트(TB) 단위의 초대용량 3D/4D 고해상도 생체 조직 스캔 데이터를 다차원 덩어리(Chunk)로 쪼개어 저장하여 웹 및 오픈소스 환경에서 빠른 액세스 지원.

### 1.2 조직 슬라이드 스캔 (병리 생체 사진) 전용 공용 포맷
1. **BigTIFF / SCN (`.scn`)**
   - **설명:** 4GB 한계를 넘는 대용량 생체 스캔 데이터를 저장하기 위해 표준 TIFF 규격을 확장한 포맷.
   - **특징:** 64비트 파일 오프셋을 지원하여 고해상도 전체 조직 슬라이드(WSI) 저장 가능.
2. **SVS (`.svs`)**
   - **설명:** 라이카 아페리오(Leica Aperio) 병리 스캐너의 표준 포맷.
   - **특징:** 피라미드 구조(Pyramid Structure)의 Multi-page TIFF 기반으로 OpenSlide 등 오픈소스 라이브러리에서 광범위하게 지원하는 사실상의 디지털 병리 표준 포맷.

---

## 2. SVS 메타데이터 추출 및 영역 타일링 (OpenSlide)

OpenSlide 라이브러리를 사용하면 파일 전체를 메모리에 올리지 않고 특정 해상도 레벨의 원하는 영역(Bounding Box)을 빠르게 잘라낼 수 있습니다.

### 2.1 Python 기본 분석 및 타일링 코드

```python
import os
import openslide

def analyze_and_extract_svs(svs_file_path, output_dir="./tiles_output"):
    if not os.path.exists(svs_file_path):
        print(f"Error: 파일을 찾을 수 없습니다 -> {svs_file_path}")
        return

    os.makedirs(output_dir, exist_ok=True)
    slide = openslide.OpenSlide(svs_file_path)

    # 1. 피라미드 레벨 및 기본 정보
    print(f"총 레벨 개수: {slide.level_count}")
    print(f"Base (Level 0) 크기 (WxH): {slide.dimensions[0]} x {slide.dimensions[1]} pixels")
    for lvl in range(slide.level_count):
        print(f" Level {lvl}: 크기 = {slide.level_dimensions[lvl]}, 축소 비율 = {slide.level_downsamples[lvl]:.2f}x")

    # 2. 주요 SVS 메타데이터
    mpp_x = slide.properties.get(openslide.PROPERTY_NAME_MPP_X, 'N/A')
    mpp_y = slide.properties.get(openslide.PROPERTY_NAME_MPP_Y, 'N/A')
    vendor = slide.properties.get(openslide.PROPERTY_NAME_VENDOR, 'N/A')
    objective = slide.properties.get('aperio.AppMag', 'N/A')

    print(f"제조사/포맷: {vendor}")
    print(f"대물렌즈 배율: {objective}x")
    print(f"Pixel Resolution (MPP X/Y): {mpp_x} / {mpp_y} um/pixel")

    # 3. 썸네일 저장
    thumbnail_img = slide.get_thumbnail((500, 500))
    thumbnail_img.save(os.path.join(output_dir, "slide_thumbnail.png"))

    # 4. 특정 영역(ROI) 타일 추출
    # read_region의 location (x, y)는 언제나 Level 0 기준 좌표임에 유의
    crop_x, crop_y = 5000, 5000
    crop_w, crop_h = 512, 512
    target_level = 0

    roi_tile = slide.read_region((crop_x, crop_y), target_level, (crop_w, crop_h)).convert("RGB")
    roi_tile.save(os.path.join(output_dir, f"roi_tile_L{target_level}_{crop_x}_{crop_y}.png"))

    slide.close()

if __name__ == "__main__":
    analyze_and_extract_svs("sample_pathology.svs")
```

---

## 3. DeepZoomGenerator를 활용한 OpenSeadragon용 DZI 변환

`openslide.deepzoom.DeepZoomGenerator`를 사용하면 웹 뷰어(OpenSeadragon)에서 다중 해상도로 탐색 가능한 DZI 피라미드 타일 폴더를 손쉽게 생성할 수 있습니다.

### 3.1 단일 프로세스 DZI 변환 코드

```python
import os
import openslide
from openslide.deepzoom import DeepZoomGenerator

def convert_svs_to_dzi(svs_path, output_prefix="output_slide", tile_size=256, overlap=1, image_format="jpeg", quality=85):
    slide = openslide.OpenSlide(svs_path)
    dz = DeepZoomGenerator(slide, tile_size=tile_size, overlap=overlap, limit_bounds=False)

    # 1. DZI XML 디스크립터 생성
    dzi_file_path = f"{output_prefix}.dzi"
    with open(dzi_file_path, "w", encoding="utf-8") as f:
        f.write(dz.get_dzi(image_format))

    # 2. 레벨별 타일 이미지 추출 및 저장
    tiles_dir = f"{output_prefix}_files"
    os.makedirs(tiles_dir, exist_ok=True)

    for level in range(dz.level_count):
        level_dir = os.path.join(tiles_dir, str(level))
        os.makedirs(level_dir, exist_ok=True)
        cols, rows = dz.level_tiles[level]

        for col in range(cols):
            for row in range(rows):
                tile_image = dz.get_tile(level, (col, row))
                tile_path = os.path.join(level_dir, f"{col}_{row}.{image_format}")
                tile_image.convert("RGB").save(tile_path, "JPEG", quality=quality)

    slide.close()

if __name__ == "__main__":
    convert_svs_to_dzi("sample_pathology.svs", "my_pathology_slide")
```

### 3.2 OpenSeadragon 웹 HTML 연동 예시

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>SVS OpenSeadragon Viewer</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/openseadragon.min.js"></script>
  <style>
    #openseadragon-viewer { width: 100vw; height: 100vh; background-color: #111; }
  </style>
</head>
<body>
  <div id="openseadragon-viewer"></div>
  <script>
    var viewer = OpenSeadragon({
      id: "openseadragon-viewer",
      prefixUrl: "https://cdnjs.cloudflare.com/ajax/libs/openseadragon/4.1.0/images/",
      tileSources: "my_pathology_slide.dzi",
      showNavigator: true,
      navigatorPosition: "BOTTOM_RIGHT"
    });
  </script>
</body>
</html>
```

---

## 4. 고속 처리를 위한 Multiprocessing 병렬 타일링

대용량 SVS 파일(수십 GB)은 타일 개수가 수만~수십만 개에 달하므로 CPU 멀티코어를 활용한 병렬 처리가 필수적입니다.

### 4.1 병렬 타일 변환 파이썬 코드

```python
import os
import time
import multiprocessing as mp
from functools import partial
import openslide
from openslide.deepzoom import DeepZoomGenerator

_slide_instance = None
_dz_instance = None

def _init_worker(svs_path, tile_size, overlap):
    global _slide_instance, _dz_instance
    _slide_instance = openslide.OpenSlide(svs_path)
    _dz_instance = DeepZoomGenerator(_slide_instance, tile_size=tile_size, overlap=overlap, limit_bounds=False)

def _process_tile_job(task, tiles_dir, image_format, quality):
    level, col, row = task
    tile_img = _dz_instance.get_tile(level, (col, row))
    
    tile_path = os.path.join(tiles_dir, str(level), f"{col}_{row}.{image_format}")
    if image_format.lower() in ["jpg", "jpeg"]:
        tile_img.convert("RGB").save(tile_path, "JPEG", quality=quality)
    else:
        tile_img.save(tile_path, "PNG")
    return 1

def parallel_convert_svs_to_dzi(
    svs_path, 
    output_prefix="output_slide", 
    tile_size=256, 
    overlap=1, 
    image_format="jpeg", 
    quality=85,
    num_workers=None
):
    if not os.path.exists(svs_path):
        print(f"[Error] SVS 파일을 찾을 수 없습니다: {svs_path}")
        return

    if num_workers is None:
        num_workers = max(1, mp.cpu_count() - 1)

    main_slide = openslide.OpenSlide(svs_path)
    main_dz = DeepZoomGenerator(main_slide, tile_size=tile_size, overlap=overlap, limit_bounds=False)

    dzi_file_path = f"{output_prefix}.dzi"
    tiles_dir = f"{output_prefix}_files"
    os.makedirs(tiles_dir, exist_ok=True)

    with open(dzi_file_path, "w", encoding="utf-8") as f:
        f.write(main_dz.get_dzi(image_format))

    tasks = []
    for level in range(main_dz.level_count):
        os.makedirs(os.path.join(tiles_dir, str(level)), exist_ok=True)
        cols, rows = main_dz.level_tiles[level]
        for col in range(cols):
            for row in range(rows):
                tasks.append((level, col, row))

    total_tiles = len(tasks)
    main_slide.close()

    worker_func = partial(_process_tile_job, tiles_dir=tiles_dir, image_format=image_format, quality=quality)

    processed_count = 0
    chunksize = max(1, min(100, total_tiles // (num_workers * 4)))

    start_time = time.time()
    with mp.Pool(processes=num_workers, initializer=_init_worker, initargs=(svs_path, tile_size, overlap)) as pool:
        for _ in pool.imap_unordered(worker_func, tasks, chunksize=chunksize):
            processed_count += 1
            if processed_count % 1000 == 0 or processed_count == total_tiles:
                elapsed = time.time() - start_time
                print(f"진행률: {(processed_count/total_tiles)*100:.1f}% [{processed_count}/{total_tiles}] | 속도: {processed_count/elapsed:.1f} tiles/sec")

if __name__ == "__main__":
    parallel_convert_svs_to_dzi("sample_pathology.svs", "fast_pathology_slide")
```

---

## 5. 핵심 요약 및 최적화 가이드

1. **`read_region()` 좌표 기준**: OpenSlide에서 `read_region((x, y), level, (w, h))` 호출 시 `(x, y)`는 지정된 `level`의 좌표가 아닌 **항상 Level 0(최고 해상도) 기준 절대 좌표**입니다.
2. **Worker 프로세스 독립 오픈**: C-라이브러리 핸들 특성상 multiprocessing 적용 시 각 Worker 프로세스가 독립적으로 `openslide.OpenSlide()`를 열도록 `initializer` 패턴을 사용해야 합니다.
3. **I/O 병목 해소**: 수만 개의 타일 이미지를 생성하므로 NVMe SSD 또는 RAM 디스크 환경에서 변환 작업을 수행하는 것이 좋습니다.
