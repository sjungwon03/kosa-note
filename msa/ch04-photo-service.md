# Chapter 4: Photo Service & 이미지 처리

## 4.1 Photo Service (사진 서비스)

### 역할
- **사진 저장**: 로컬 파일 시스템에 사진 저장
- **사진 조회**: object_key로 사진 반환
- **사진 삭제**: 직원 삭제 시 연관 사진 삭제

---

## 4.2 Photo Service 구현

### 기본 설정
```python
import os
import shutil
import uuid
from fastapi import FastAPI, UploadFile, File, HTTPException
from fastapi.responses import FileResponse, JSONResponse

PHOTOS_DIR = "data/photos"
os.makedirs(PHOTOS_DIR, exist_ok=True)

app = FastAPI()
```

### 사진 업로드
```python
@app.post("/upload")
async def upload_photo(file: UploadFile = File(...)):
    if not file.filename:
        raise HTTPException(status_code=400, detail="No file selected")

    # 고유 파일명 생성 (UUID)
    file_extension = file.filename.split(".")[-1] if "." in file.filename else "bin"
    object_key = f"{uuid.uuid4()}.{file_extension}"
    file_path = os.path.join(PHOTOS_DIR, object_key)

    try:
        with open(file_path, "wb") as buffer:
            shutil.copyfileobj(file.file, buffer)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Could not upload file: {e}")

    return JSONResponse(status_code=200, content={"object_key": object_key})
```

### 사진 조회
```python
@app.get("/photos/{object_key}")
async def get_photo(object_key: str):
    file_path = os.path.join(PHOTOS_DIR, object_key)
    if not os.path.exists(file_path):
        raise HTTPException(status_code=404, detail="Photo not found")
    
    return FileResponse(file_path)
```

### 사진 삭제
```python
@app.delete("/photos/{object_key}")
async def delete_photo(object_key: str):
    file_path = os.path.join(PHOTOS_DIR, object_key)
    if not os.path.exists(file_path):
        raise HTTPException(status_code=404, detail="Photo not found")
    
    try:
        os.remove(file_path)
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Could not delete file: {e}")
    
    return JSONResponse(status_code=200, content={"message": f"Photo {object_key} deleted."})
```

---

## 4.3 이미지 처리 (util.py)

### resize_image 함수
```python
from io import BytesIO
from PIL import Image

EXIF_ORIENTATION = 274

def resize_image(file_p, size):
    dest_ratio = size[0] / float(size[1])
    
    # 이미지 열기
    image = Image.open(file_p)
    
    # EXIF orientation 처리 (회전 보정)
    try:
        exif = dict(image._getexif().items())
        if exif[EXIF_ORIENTATION] == 3:
            image = image.rotate(180, expand=True)
        elif exif[EXIF_ORIENTATION] == 6:
            image = image.rotate(270, expand=True)
        elif exif[EXIF_ORIENTATION] == 8:
            image = image.rotate(90, expand=True)
    except:
        pass
    
    # 비율 계산
    source_ratio = image.size[0] / float(image.size[1])
    
    # 리사이즈
    if image.size < size:
        new_width, new_height = image.size
    elif dest_ratio > source_ratio:
        new_width = int(image.size[0] * size[1] / float(image.size[1]))
        new_height = size[1]
    else:
        new_width = size[0]
        new_height = int(image.size[1] * size[0] / float(image.size[0]))
    
    image = image.resize((new_width, new_height), resample=Image.LANCZOS)
    
    # 최종 이미지 생성
    final_image = Image.new("RGBA", size)
    topleft = (int((size[0] - new_width) / float(2)),
               int((size[1] - new_height) / float(2)))
    final_image.paste(image, topleft)
    
    # Bytes 반환
    bytes_stream = BytesIO()
    final_image.save(bytes_stream, 'PNG')
    return bytes_stream.getvalue()
```

### 리사이즈 크기
- **기본**: (120, 160) pixels
- 중앙 정렬, 비율 유지

---

## 4.4 Employee Server와 연동

### 사진 업로드 요청
```python
# Employee Server → Photo Service
image_bytes = util.resize_image(photo.file, (120, 160))

files = {'file': (photo.filename, image_bytes, photo.content_type)}
response = await client.post(f"{PHOTO_SERVICE_URL}/upload", files=files)
upload_result = response.json()
key = upload_result.get("object_key")
```

### 사진 URL 구성
```python
def get_photo_url_for_fastapi(object_key: str):
    return f"/uploads/{object_key}"
```

---

## 4.5 S3 모방 구조

### Local File System
```
photo_service/
├── app.py
├── data/
│   └── photos/
│       ├── uuid1.png
│       ├── uuid2.jpg
│       └── ...
```

### object_key 예시
```
550e8400-e29b-41d4-a716-446655440000.png
```