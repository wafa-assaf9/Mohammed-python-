import os
import numpy as np
import matplotlib.pyplot as plt
from PIL import Image, ImageOps, ImageFilter, ImageDraw

# ============ الإعدادات ============
IMAGE_PATH = "photo.jfif"  # يفضل استخدام jpg أو png
OUTPUT_PATH = "pop_art_result.png"
PANEL_SIZE = 500  # حجم كل مربع بالبكسل
MODE = "warhol"  # "warhol" أو "halftone"

# ألوان كل لوحة (من الأغمق إلى الأفتح)
PALETTES = [
    [(25, 25, 70), (230, 30, 100), (255, 200, 0), (255, 250, 235)],
    [(10, 60, 90), (0, 170, 190), (255, 90, 60), (255, 245, 200)],
    [(60, 0, 90), (150, 40, 200), (255, 120, 180), (255, 255, 120)],
    [(20, 20, 20), (220, 30, 30), (40, 160, 90), (250, 235, 60)],
]


# ============ تجهيز الصورة ============
def load_gray(path, size):
    """يفتح الصورة، يقصّها مربعة، يحوّلها رمادي ويقوّي التباين."""
    if not os.path.exists(path):
        raise FileNotFoundError(f"لم يتم العثور على الملف: {path}. تأكدي من وجود الصورة في نفس المجلد.")

    img = Image.open(path).convert("RGB")
    img = ImageOps.fit(img, (size, size), centering=(0.5, 0.35))  # التركيز على الوجه
    gray = ImageOps.grayscale(img)
    gray = ImageOps.autocontrast(gray, cutoff=3)
    gray = gray.filter(ImageFilter.GaussianBlur(1.0))  # تقليل بسيط للتشويش
    return gray


# ============ الأسلوب الأول: Warhol (4 لوحات ملونة) ============
def colorize(gray_img, palette):
    """يقسّم درجات الرمادي إلى مستويات ثابته ليعطي تباينPop Art قوي."""
    arr = np.array(gray_img)
    levels = len(palette)

    # تقسيم نطاق 0-255 إلى أجزاء متساوية لتبدو الخطوط حادة
    idx = (arr // (256 // levels)).clip(0, levels - 1)

    colored = np.array(palette, dtype=np.uint8)[idx]
    return Image.fromarray(colored, "RGB")


def warhol(gray_img):
    panels = [colorize(gray_img, p) for p in PALETTES]
    w, h = gray_img.size
    canvas = Image.new("RGB", (w * 2, h * 2))
    positions = [(0, 0), (w, 0), (0, h), (w, h)]
    for panel, pos in zip(panels, positions):
        canvas.paste(panel, pos)
    return canvas


# ============ الأسلوب الثاني: Halftone (نقاط Lichtenstein) ============
def halftone(gray_img, dot_step=10, dot_color=(220, 30, 60), bg_color=(255, 235, 120)):
    w, h = gray_img.size
    scale = 4
    canvas = Image.new("RGB", (w * scale, h * scale), bg_color)
    draw = ImageDraw.Draw(canvas)
    arr = np.array(gray_img)

    for y in range(0, h, dot_step):
        for x in range(0, w, dot_step):
            block = arr[y:y + dot_step, x:x + dot_step]
            if block.size == 0:
                continue
            darkness = 1 - block.mean() / 255
            r = darkness * dot_step * 0.75 * scale
            cx = (x + dot_step / 2) * scale
            cy = (y + dot_step / 2) * scale
            draw.ellipse([cx - r, cy - r, cx + r, cy + r], fill=dot_color)

    return canvas.resize((w, h), Image.LANCZOS)


# ============ التشغيل ============
if name == "main":
    try:
        gray = load_gray(IMAGE_PATH, PANEL_SIZE)

        if MODE == "warhol":
            result = warhol(gray)
        else:
            result = halftone(gray)

        result.save(OUTPUT_PATH)
        print(f"تم الحفظ بنجاح في: {OUTPUT_PATH}")

        plt.figure(figsize=(8, 8))
        plt.imshow(result)
        plt.axis("off")
        plt.show()
    except Exception as e:
        print(f"حدث خطأ أثناء التشغيل: {e}")
