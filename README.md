import os
import shutil
import ffmpeg
import cv2
import numpy as np
import stripe
from fastapi import FastAPI, UploadFile, File, Form
from fastapi.responses import FileResponse
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

TEMP_DIR = "temp"
os.makedirs(TEMP_DIR, exist_ok=True)

stripe.api_key = os.getenv("STRIPE_SECRET_KEY")

# ----------------------------
# SIMPLE TRANSITION FUNCTION
# ----------------------------

def simple_blend(video1, video2):
    cap1 = cv2.VideoCapture(video1)
    cap2 = cv2.VideoCapture(video2)

    output_path = os.path.join(TEMP_DIR, "transition.mp4")

    fourcc = cv2.VideoWriter_fourcc(*"mp4v")
    fps = int(cap1.get(cv2.CAP_PROP_FPS))
    width = int(cap1.get(cv2.CAP_PROP_FRAME_WIDTH))
    height = int(cap1.get(cv2.CAP_PROP_FRAME_HEIGHT))

    out = cv2.VideoWriter(output_path, fourcc, fps, (width, height))

    frames1, frames2 = [], []

    while True:
        ret, frame = cap1.read()
        if not ret:
            break
        frames1.append(frame)

    while True:
        ret, frame = cap2.read()
        if not ret:
            break
        frames2.append(frame)

    min_len = min(len(frames1), len(frames2))

    for i in range(min_len):
        alpha = i / min_len
        blended = cv2.addWeighted(frames1[i], 1 - alpha, frames2[i], alpha, 0)
        out.write(blended)

    cap1.release()
    cap2.release()
    out.release()

    return output_path


# ----------------------------
# TRANSITION ENDPOINT
# ----------------------------

@app.post("/generate-transition")
async def generate_transition(
    clipA: UploadFile = File(...),
    clipB: UploadFile = File(...)
):
    pathA = os.path.join(TEMP_DIR, "clipA.mp4")
    pathB = os.path.join(TEMP_DIR, "clipB.mp4")

    with open(pathA, "wb") as buffer:
        shutil.copyfileobj(clipA.file, buffer)

    with open(pathB, "wb") as buffer:
        shutil.copyfileobj(clipB.file, buffer)

    transition = simple_blend(pathA, pathB)

    return FileResponse(transition, media_type="video/mp4")


# ----------------------------
# STRIPE CHECKOUT
# ----------------------------

@app.post("/create-checkout")
def create_checkout():
    session = stripe.checkout.Session.create(
        payment_method_types=["card"],
        mode="subscription",
        line_items=[{
            "price": os.getenv("STRIPE_PRICE_ID"),
            "quantity": 1,
        }],
        success_url="https://yourdomain.com/success",
        cancel_url="https://yourdomain.com/cancel",
    )

    return {"checkout_url": session.url}
