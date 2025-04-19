fastapi
uvicorn
pydantic
motor         # async MongoDB driver
python-jose   # JWT tokens
bcrypt
openai
python-dotenv
# Python
__pycache__/
*.py[cod]
*.egg-info/
*.env

# Environment files
.env

# OS-specific
.DS_Store
Thumbs.db

# IDEs/editors
.vscode/
.idea/

# Virtual environments
venv/
.env/
env/
import os
from dotenv import load_dotenv

load_dotenv()

OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
SECRET_KEY = os.getenv("SECRET_KEY")
MONGO_URI = os.getenv("MONGO_URI")
from pydantic import BaseModel, EmailStr

class User(BaseModel):
    email: EmailStr
    password: str

class UserInDB(User):
    hashed_password: str
from motor.motor_asyncio import AsyncIOMotorClient
from config import MONGO_URI

client = AsyncIOMotorClient(MONGO_URI)
db = client.chatbot
from passlib.context import CryptContext
from jose import jwt
from config import SECRET_KEY

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password):
    return pwd_context.hash(password)

def verify_password(plain, hashed):
    return pwd_context.verify(plain, hashed)

def create_jwt(email):
    return jwt.encode({"sub": email}, SECRET_KEY, algorithm="HS256")
from fastapi import APIRouter, HTTPException
from models.user import User
from database import db
from utils.auth import hash_password, verify_password, create_jwt

router = APIRouter()

@router.post("/register")
async def register(user: User):
    existing = await db.users.find_one({"email": user.email})
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    hashed = hash_password(user.password)
    await db.users.insert_one({"email": user.email, "hashed_password": hashed})
    return {"msg": "User registered"}

@router.post("/login")
async def login(user: User):
    record = await db.users.find_one({"email": user.email})
    if not record or not verify_password(user.password, record["hashed_password"]):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    token = create_jwt(user.email)
    return {"access_token": token}
from fastapi import APIRouter, Body
import openai
from config import OPENAI_API_KEY

openai.api_key = OPENAI_API_KEY
router = APIRouter()

@router.post("/chat")
async def ask_gpt(prompt: str = Body(...)):
    res = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return {"response": res['choices'][0]['message']['content']}

@router.post("/image")
async def generate_image(prompt: str = Body(...)):
    res = openai.Image.create(prompt=prompt, n=1, size="512x512")
    return {"image_url": res['data'][0]['url']}
from fastapi import FastAPI
from routes import auth, chat

app = FastAPI()
app.include_router(auth.router, prefix="/auth")
app.include_router(chat.router, prefix="/ai")
