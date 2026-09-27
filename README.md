# ai-video-maker
Tạo video AI art với chữ 3D, hiệu ứng chuyển động, và nhạc nền
import os
import glob
import math
import random
import subprocess

import cv2
import numpy as np
from PIL import Image, ImageDraw, ImageFont, ImageFilter


# =====================================================
# CẤU HÌNH
# =====================================================

WIDTH = 1280
HEIGHT = 720
FPS = 30
SECONDS_PER_IMAGE = 4

OUTPUT_TEMP = "video_without_music.mp4"
OUTPUT_FINAL = "ai_art_advertising.mp4"

IMAGE_FOLDER = "ai_images"
MUSIC_FILE = "music.mp3"

# Nội dung chữ quảng cáo
MAIN_TEXT = "AI FUTURE"
SUB_TEXT = "CREATE • IMAGINE • INSPIRE"

# Font chữ
FONT_PATH = "DejaVuSans-Bold.ttf"

# Nếu dùng Windows, có thể dùng:
# FONT_PATH = "C:/Windows/Fonts/arialbd.ttf"


# =====================================================
# HÀM TIỆN ÍCH
# =====================================================

def load_font(size):
    """
    Tải font chữ.
    """
    try:
        return ImageFont.truetype(FONT_PATH, size)
    except:
        print("Không tìm thấy font, sử dụng font mặc định.")
        return ImageFont.load_default()


def fit_image_to_screen(image, width, height, zoom=1.0):
    """
    Cắt ảnh theo tỷ lệ màn hình 16:9 và phóng to nhẹ.
    """
    image = image.convert("RGB")

    image_ratio = image.width / image.height
    screen_ratio = width / height

    if image_ratio > screen_ratio:
        # Ảnh quá rộng
        new_height = height
        new_width = int(new_height * image_ratio)
    else:
        # Ảnh quá cao
        new_width = width
        new_height = int(new_width / image_ratio)

    image = image.resize((new_width, new_height), Image.Resampling.LANCZOS)

    # Phóng to tạo hiệu ứng camera tiến vào ảnh
    zoom_width = int(new_width * zoom)
    zoom_height = int(new_height * zoom)
    image = image.resize((zoom_width, zoom_height), Image.Resampling.LANCZOS)

    # Tọa độ crop có chuyển động nhẹ
    max_x = max(0, image.width - width)
    max_y = max(0, image.height - height)

    return image, max_x, max_y


def add_color_overlay(image, t):
    """
    Thêm lớp màu chuyển động lên ảnh AI.
    """
    overlay = Image.new("RGBA", image.size, (0, 0, 0, 0))
    draw = ImageDraw.Draw(overlay)

    w, h = image.size

    # Quầng sáng chuyển động
    x = int(w * (0.2 + 0.6 * ((math.sin(t * 0.8) + 1) / 2)))
    y = int(h * (0.25 + 0.5 * ((math.cos(t * 0.6) + 1) / 2)))

    for radius in range(300, 20, -25):
        alpha = max(0, int(70 * (1 - radius / 330)))
        draw.ellipse(
            (
                x - radius,
                y - radius,
                x + radius,
                y + radius
            ),
            fill=(100, 30, 255, alpha)
        )

    # Dải màu xanh
    strip_x = int((math.sin(t * 1.5) + 1) / 2 * w)
    draw.rectangle(
        (strip_x - 80, 0, strip_x + 80, h),
        fill=(0, 210, 255, 30)
    )

    overlay = overlay.filter(ImageFilter.GaussianBlur(35))
    return Image.alpha_composite(image.convert("RGBA"), overlay)


def draw_3d_text(base, text, center_x, center_y, size, t):
    """
    Vẽ chữ lớn dạng 3D, có bóng đổ và ánh sáng.
    """
    draw = ImageDraw.Draw(base)
    font = load_font(size)

    box = draw.textbbox((0, 0), text, font=font, stroke_width=0)
    text_width = box[2] - box[0]
    text_height = box[3] - box[1]

    x = int(center_x - text_width / 2)
    y = int(center_y - text_height / 2)

    # Lớp bóng mờ phát sáng
    glow = Image.new("RGBA", base.size, (0, 0, 0, 0))
    glow_draw = ImageDraw.Draw(glow)

    glow_draw.text(
        (x, y),
        text,
        font=font,
        fill=(0, 190, 255, 130),
        stroke_width=18,
        stroke_fill=(0, 80, 255, 100)
    )

    glow = glow.filter(ImageFilter.GaussianBlur(18))
    base.alpha_composite(glow)

    # Các lớp tạo độ sâu 3D
    for depth in range(22, 0, -1):
        offset_x = depth * 2
        offset_y = depth * 2

        depth_color = (
            max(0, 18 - depth),
            max(0, 35 - depth),
            min(255, 80 + depth * 4),
            255
        )

        draw.text(
            (x + offset_x, y + offset_y),
            text,
            font=font,
            fill=depth_color,
            stroke_width=5,
            stroke_fill=(5, 10, 35, 255)
        )

    # Màu chữ chính
    color_shift = int((math.sin(t * 2) + 1) * 40)

    draw.text(
        (x, y),
        text,
        font=font,
        fill=(255, 220, 100 + color_shift, 255),
        stroke_width=5,
        stroke_fill=(255, 255, 255, 255)
    )

    # Highlight chạy trên chữ
    highlight_x = int(x + (text_width + 100) * ((t * 0.6) % 1.2) - 100)

    highlight = Image.new("RGBA", base.size, (0, 0, 0, 0))
    highlight_draw = ImageDraw.Draw(highlight)

    highlight_draw.rectangle(
        (highlight_x, y - 20, highlight_x + 100, y + text_height + 20),
        fill=(255, 255, 255, 80)
    )

    highlight = highlight.filter(ImageFilter.GaussianBlur(20))

    # Chỉ tạo hiệu ứng ánh sáng chung, không ảnh hưởng nhiều đến nền
    base.alpha_composite(highlight)


def draw_scrolling_text(base, text, y, t, speed=180):
    """
    Vẽ chữ chạy ngang như banner quảng cáo.
    """
    draw = ImageDraw.Draw(base)
    font = load_font(32)

    box = draw.textbbox((0, 0), text, font=font)
    text_width = box[2] - box[0]

    # Chạy từ phải sang trái
    x = int(WIDTH - ((t * speed) % (WIDTH + text_width)))

    # Nền banner bán trong suốt
    banner = Image.new("RGBA", (WIDTH, 65), (0, 0, 0, 120))
    banner_draw = ImageDraw.Draw(banner)

    banner_draw.line(
        (0, 3, WIDTH, 3),
        fill=(0, 220, 255, 220),
        width=3
    )

    banner_draw.line(
        (0, 61, WIDTH, 61),
        fill=(255, 80, 180, 220),
        width=3
    )

    base.alpha_composite(banner, (0, y - 15))

    draw = ImageDraw.Draw(base)

    draw.text(
        (x + 3, y + 3),
        text,
        font=font,
        fill=(0, 0, 0, 220)
    )

    draw.text(
        (x, y),
        text,
        font=font,
        fill=(255, 255, 255, 255)
    )


def draw_particles(base, t, seed=10):
    """
    Tạo các hạt ánh sáng chuyển động.
    """
    random.seed(seed)
    draw = ImageDraw.Draw(base)

    for i in range(100):
        start_x = random.randint(0, WIDTH)
        start_y = random.randint(0, HEIGHT)

        x = int(start_x + math.sin(t * 1.5 + i) * 40)
        y = int(start_y - (t * 35 + i * 5) % HEIGHT)

        radius = random.choice([1, 1, 2, 3])
        color = random.choice([
            (0, 220, 255, 170),
            (255, 100, 220, 160),
            (255, 230, 100, 180),
            (255, 255, 255, 150)
        ])

        draw.ellipse(
            (x - radius, y - radius, x + radius, y + radius),
            fill=color
        )


# =====================================================
# TẠO VIDEO
# =====================================================

def create_video():
    image_files = []

    for extension in ["*.jpg", "*.jpeg", "*.png", "*.webp"]:
        image_files.extend(
            glob.glob(os.path.join(IMAGE_FOLDER, extension))
        )

    image_files.sort()

    if not image_files:
        raise FileNotFoundError(
            f"Không tìm thấy ảnh trong thư mục: {IMAGE_FOLDER}"
        )

    print(f"Đã tìm thấy {len(image_files)} ảnh.")

    total_frames = len(image_files) * SECONDS_PER_IMAGE * FPS

    fourcc = cv2.VideoWriter_fourcc(*"mp4v")
    writer = cv2.VideoWriter(
        OUTPUT_TEMP,
        fourcc,
        FPS,
        (WIDTH, HEIGHT)
    )

    for frame_number in range(total_frames):
        current_time = frame_number / FPS

        image_index = int(
            current_time // SECONDS_PER_IMAGE
        ) % len(image_files)

        time_in_image = current_time % SECONDS_PER_IMAGE
        progress = time_in_image / SECONDS_PER_IMAGE

        # Đọc ảnh AI
        image = Image.open(image_files[image_index])

        # Zoom từ 1.00 đến 1.15
        zoom = 1.0 + progress * 0.15

        image, max_x, max_y = fit_image_to_screen(
            image,
            WIDTH,
            HEIGHT,
            zoom
        )

        # Camera di chuyển chậm trong ảnh
        crop_x = int(
            max_x * (0.5 + 0.5 * math.sin(progress * math.pi))
        )
        crop_y = int(
            max_y * (0.5 + 0.5 * math.cos(progress * math.pi))
        )

        image = image.crop(
            (crop_x, crop_y, crop_x + WIDTH, crop_y + HEIGHT)
        )

        # Hiệu ứng màu
        image = add_color_overlay(image, current_time)

        # Lớp tối nhẹ để chữ nổi bật
        dark_layer = Image.new(
            "RGBA",
            (WIDTH, HEIGHT),
            (0, 0, 30, 65)
        )
        image = Image.alpha_composite(image, dark_layer)

        # Hạt ánh sáng
        draw_particles(image, current_time, image_index + 50)

        # Chữ chính chuyển động nhẹ
        text_x = WIDTH / 2 + math.sin(current_time * 1.4) * 35
        text_y = HEIGHT * 0.48 + math.cos(current_time * 1.1) * 15

        draw_3d_text(
            image,
            MAIN_TEXT,
            text_x,
            text_y,
            115,
            current_time
        )

        # Chữ phụ
        draw = ImageDraw.Draw(image)
        sub_font = load_font(28)

        sub_box = draw.textbbox(
            (0, 0),
            SUB_TEXT,
            font=sub_font
        )

        sub_width = sub_box[2] - sub_box[0]

        draw.text(
            (
                int((WIDTH - sub_width) / 2),
                HEIGHT // 2 + 95
            ),
            SUB_TEXT,
            font=sub_font,
            fill=(255, 255, 255, 240),
            stroke_width=1,
            stroke_fill=(0, 0, 0, 255)
        )

        # Chữ chạy phía dưới
        draw_scrolling_text(
            image,
            "  AI ART  •  FUTURE DESIGN  •  CREATE YOUR WORLD  •  ",
            HEIGHT - 60,
            current_time,
            speed=220
        )

        # Chuyển PIL sang OpenCV
        frame_rgb = np.array(image.convert("RGB"))
        frame_bgr = cv2.cvtColor(frame_rgb, cv2.COLOR_RGB2BGR)

        writer.write(frame_bgr)

        if frame_number % FPS == 0:
            print(
                f"Đang tạo video: "
                f"{frame_number / total_frames * 100:.1f}%"
            )

    writer.release()
    print(f"Đã tạo video hình ảnh: {OUTPUT_TEMP}")


# =====================================================
# GHÉP NHẠC BẰNG FFMPEG
# =====================================================

def add_music():
    if not os.path.exists(MUSIC_FILE):
        print("Không tìm thấy music.mp3.")
        print(f"Video chưa có nhạc: {OUTPUT_TEMP}")
        return

    command = [
        "ffmpeg",
        "-y",
        "-i", OUTPUT_TEMP,
        "-stream_loop", "-1",
        "-i", MUSIC_FILE,
        "-map", "0:v:0",
        "-map", "1:a:0",
        "-c:v", "copy",
        "-c:a", "aac",
        "-shortest",
        OUTPUT_FINAL
    ]

    try:
        subprocess.run(command, check=True)
        print(f"Đã hoàn thành video: {OUTPUT_FINAL}")
    except FileNotFoundError:
        print("Chưa cài FFmpeg.")
        print(f"Video không có nhạc được lưu tại: {OUTPUT_TEMP}")
    except subprocess.CalledProcessError:
        print("Lỗi khi ghép nhạc bằng FFmpeg.")


if __name__ == "__main__":
    create_video()
    add_music()
