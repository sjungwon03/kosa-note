# Chapter 3: Auth Server & Employee Server

## 3.1 Auth Server (인증 서버)

### 역할
- **JWT Token 발급**: 로그인 시 JWT 생성
- **Token 검증**: Employee Server에서 토큰 검증

---

## 3.2 Auth Server 구현

### 기본 설정
```python
import jwt
import datetime
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

SECRET_KEY = 'your-super-secret-key-change-it'

class LoginRequest(BaseModel):
    username: str
    password: str
```

### 로그인 처리
```python
@app.post('/login')
async def login(user_credentials: LoginRequest):
    username = user_credentials.username
    password = user_credentials.password

    # 하드코딩된 인증 (실제는 DB 조회)
    if username == 'admin' and password == 'password':
        token = jwt.encode({
            'user': username,
            'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=1)
        }, SECRET_KEY, algorithm="HS256")
        return {'token': token}

    raise HTTPException(status_code=401, detail="Invalid credentials")
```

### JWT Token 구조
```json
{
  "user": "admin",
  "exp": 1699999999  // 1시간 후 만료
}
```

---

## 3.3 Employee Server (직원 서버)

### 역할
- **CRUD**: 직원 정보 생성, 조회, 수정, 삭제
- **JWT 인증**: 모든 요청에 토큰 검증
- **Photo 연동**: 사진 서비스로 업로드 요청

---

## 3.4 JWT 인증 구현

### OAuth2 Bearer 설정
```python
from fastapi.security import OAuth2PasswordBearer

SECRET_KEY = config.JWT_SECRET_KEY
ALGORITHM = "HS256"

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("user")
        if username is None:
            raise credentials_exception
        return username
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token has expired")
    except jwt.PyJWTError:
        raise credentials_exception
```

### 인증 적용
```python
@app.get("/employees")
async def get_employees(current_user: str = Depends(get_current_user)):
    # current_user에 인증된 사용자명 저장
    employees = database.list_employees()
    return employees
```

---

## 3.5 CRUD API

### 직원 목록 조회
```python
@app.get("/employees", response_model=EmployeesListResponse)
async def get_employees(current_user: str = Depends(get_current_user)):
    employees: List[Employee] = database.list_employees()
    
    employees_public_data = []
    for employee in employees:
        emp_public = EmployeePublic.from_orm(employee)
        if employee.object_key:
            emp_public.photo_url = f"/uploads/{employee.object_key}"
        employees_public_data.append(emp_public)
    
    return employees_public_data
```

### 직원 단건 조회
```python
@app.get("/employee/{employee_id}", response_model=EmployeePublic)
async def get_employee(employee_id: int, current_user: str = Depends(get_current_user)):
    employee = database.load_employee(employee_id)
    if employee:
        emp_public = EmployeePublic.from_orm(employee)
        if employee.object_key:
            emp_public.photo_url = f"/uploads/{employee.object_key}"
        return emp_public
    raise HTTPException(status_code=404, detail="Employee not found")
```

### 직원 생성/수정
```python
@app.post("/employee", response_model=Employee)
async def save_employee(
    full_name: str = Form(...),
    location: str = Form(...),
    job_title: str = Form(...),
    badges: str = Form(""),
    employee_id: Optional[int] = Form(None),
    photo: Optional[UploadFile] = File(None),
    current_user: str = Depends(get_current_user)
):
    key = None
    if photo and photo.filename != '':
        image_bytes = util.resize_image(photo.file, (120, 160))
        if image_bytes:
            # Photo Service에 업로드
            files = {'file': (photo.filename, image_bytes, photo.content_type)}
            response = await client.post(f"{config.PHOTO_SERVICE_URL}/upload", files=files)
            upload_result = response.json()
            key = upload_result.get("object_key")
    
    employee_data = Employee(
        id=employee_id,
        object_key=key,
        full_name=full_name,
        location=location,
        job_title=job_title,
        badges=badges
    )
    
    if employee_id:
        # 이전 사진 삭제 후 업데이트
        old_employee = database.load_employee(employee_id)
        if old_employee and old_employee.object_key:
            await client.delete(f"{config.PHOTO_SERVICE_URL}/photos/{old_employee.object_key}")
        return database.update_employee(employee_id, employee_data)
    else:
        return database.add_employee(employee_data)
```

### 직원 삭제
```python
@app.delete("/employee/{employee_id}")
async def delete_employee_route(employee_id: int, current_user: str = Depends(get_current_user)):
    employee = database.load_employee(employee_id)
    if not employee:
        raise HTTPException(status_code=404, detail="Employee not found")
    
    # Photo Service에서 사진 삭제
    if employee.object_key:
        await client.delete(f"{config.PHOTO_SERVICE_URL}/photos/{employee.object_key}")
    
    database.delete_employee(employee_id)
    return JSONResponse(status_code=200, content={"success": True})
```

---

## 3.6 데이터베이스 모드

### RDS 모드
```python
import database  # database.py (MySQL)
```

### DynamoDB 모드
```python
if "DYNAMO_MODE" in os.environ:
    import database_dynamo as database
```

---

## 3.7 Pydantic 모델

### Employee 모델
```python
from pydantic import BaseModel
from typing import Optional

class Employee(BaseModel):
    id: Optional[int]
    object_key: Optional[str]
    full_name: str
    location: str
    job_title: str
    badges: Optional[str]

class EmployeePublic(BaseModel):
    id: int
    full_name: str
    location: str
    job_title: str
    badges: Optional[str]
    photo_url: Optional[str]
```