# Copilot Chat Conversation Export: Telegram bot for video downloading

**User:** @mohamedbarakat22334455-cpu
**Thread URL:** https://github.com/copilot/c/7c907ef1-eba8-4818-ab72-59ba4f602062

## @mohamedbarakat22334455-cpu

import os
import yt_dlp
from aiogram import Bot, Dispatcher, types
from aiogram.utils import executor

# --- الإعدادات ---
# حط التوكن بتاعك هنا بين العلامتين ''
API_TOKEN = 'هنا_تحط_التوكن_اللي_من_BotFather'
MY_WALLET = 'UQBS4-2MIVdF1jDxJK0YkDO5z8Y-5vHIk3gQwzZ7egE1PC_'

bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot)

@dp.message_handler(commands=['start'])
async def welcome(message: types.Message):
    await message.reply(
        f"🔥 أهلاً بك في منصة MB للتحميل!\n\n"
        f"فقط أرسل رابط الفيديو (TikTok, IG, YouTube) وسأقوم بجلب النسخة الأصلية لك.\n\n"
        f"💎 لدعم المطور (TON):\n`{MY_WALLET}`"
    )

@dp.message_handler()
async def handle_download(message: types.Message):
    url = message.text
    if "http" in url:
        sent_msg = await message.answer("⏳ جاري المعالجة... انتظر ثواني")
        
        # إعدادات التحميل (بدون علامة مائية وأفضل جودة)
        ydl_opts = {
            'format': 'best',
            'outtmpl': 'video.mp4',
            'quiet': True,
            'no_warnings': True,
        }

        try:
            with yt_dlp.YoutubeDL(ydl_opts) as ydl:
                ydl.download([url])
            
            # إرسال الفيديو
            with open('video.mp4', 'rb') as video:
                await bot.send_video(message.chat.id, video, caption="✅ تم التحميل بواسطة MB")
            
            # تنظيف السيرفر
            os.remove('video.mp4')
            await bot.delete_message(message.chat.id, sent_msg.message_id)
            
        except Exception as e:
            await message.reply("❌ عذراً، هذا الرابط غير مدعوم أو فيه مشكلة.")
            if os.path.exists("video.mp4"):
                os.remove("video.mp4")

if __name__ == '__main__':
    print("البوت شغال دلوقتي.. روح جربه في تليجرام!")
    executor.start_polling(dp, skip_updates=True)

## Copilot

I can see you've shared a Telegram bot script that uses `yt_dlp` to download videos from various platforms. However, I notice this appears to be code shared directly in chat rather than from a GitHub repository context.

Before I can provide meaningful assistance, I'd like to clarify:

**What would you like help with regarding this code?** For example:
- Do you want to fix a bug or error you're encountering?
- Are you looking to add new features?
- Do you need help understanding how a specific part works?
- Are you trying to deploy this somewhere?
- Do you want security or best practice improvements?

**Also, a few important notes:**

⚠️ **Security & Legal Considerations:**
- The hardcoded API token placeholder should never be committed to public repositories
- Using bots to mass-download content from platforms may violate their Terms of Service
- Some platforms explicitly prohibit automated downloads of their content
- Consider the copyright status of content being downloaded

**If you'd like to work with this in GitHub:**
- Would you like help creating a GitHub issue for a specific problem?
- Do you want to set this up as a proper GitHub repository?
- Need help with configuration management (environment variables instead of hardcoded tokens)?

Please let me know what specific help you need, and I'll be better able to assist you!

*System: User retried message*

## Copilot

أرسلت شفرة البوت بدون توضيح المطلوب—فهمت إنك قد تريد تحسينها أو جعلها أكثر أمانًا ومرونة. أعطيت أدناه ملاحظات سريعة ثم نسخة محسّنة من الكود (تتعامل مع التحميل بشكل آمن غير متزامن، أسماء ملفات مؤقتة، حدود تزامن، وإخراج أفضل للأخطاء). لو تريد أن أعدل شيء محدد (مثلاً: دعم Watermark removal خاص لـ TikTok، رفع الملفات الكبيرة لــ Dropbox/S3، ملف Docker، أو نشر على VPS) قل لي.

ملاحظات مهمة قبل الاستخدام
- لا تترك توكن البوت (API_TOKEN) مكتوبًا في الكود. خزّنه في متغيّات بيئة (ENV).
- أيضًا فكّر بإخفاء عنوان المحفظة إذا لا تريد نشره، أو احمله من ENV.
- yt_dlp عملية حظريّة — لا يجب تشغيلها مباشرة في حلقة asyncio؛ الشيفرة المحسّنة تستخدم run_in_executor.
- استخدم مسار مؤقت فريد لكل تحميل لتجنب تعارضات متزامنة.
- تحقق من حجم الملف قبل الإرسال. إذا كان أكبر من حد Telegram للـ bot API قد تحتاج حلول بديلة (رفع إلى تخزين سحابي ثم إرسال الرابط، أو استخدام userbot).
- سجّل الأخطاء (logging) بدل طبعها للعامة.

النسخة المحسّنة (async-safe، مؤقتة، حدّ لتزامن التحميلات)
```python name=bot_improved.py
import os
import asyncio
import shutil
import logging
import tempfile
from pathlib import Path
import yt_dlp
from aiogram import Bot, Dispatcher, types
from aiogram.utils import executor

# Config from environment (recommended)
API_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "هنا_تحط_التوكن_اللي_من_BotFather")
MY_WALLET = os.getenv("MY_WALLET", "UQBS4-...")

if "هنا_تحط_التوكن_اللي_من_BotFather" in API_TOKEN:
    logging.warning("Please set TELEGRAM_BOT_TOKEN environment variable and avoid committing the token in code.")

# Basic logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

bot = Bot(token=API_TOKEN)
dp = Dispatcher(bot)

# Limit concurrent downloads to avoid overload
DOWNLOAD_SEMAPHORE = asyncio.Semaphore(2)

# Size limit advice (adjust if you know your Bot API limits or use external upload)
TG_SAFE_SIZE_BYTES = 50 * 1024 * 1024  # 50 MB suggested fallback

@dp.message_handler(commands=['start'])
async def welcome(message: types.Message):
    await message.reply(
        "🔥 أهلاً بك! أرسل رابط فيديو (TikTok, IG, YouTube) وسأحاول تحميله.\n\n"
        f"💎 دعم (TON): `{MY_WALLET}`"
    )

def run_ydl_sync(url: str, outtmpl_dir: str) -> str:
    """
    Runs yt_dlp synchronously (to be called in executor).
    Returns the path of downloaded file (first matching file) or raises.
    """
    ydl_opts = {
        'format': 'bestvideo+bestaudio/best',
        'outtmpl': os.path.join(outtmpl_dir, '%(id)s.%(ext)s'),
        'merge_output_format': 'mp4',
        'quiet': True,
        'no_warnings': True,
        # 'noplaylist': True,  # enable if you want to avoid playlists by default
    }

    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        ydl.download([url])

    # find downloaded file
    files = list(Path(outtmpl_dir).glob("*"))
    if not files:
        raise FileNotFoundError("No file was downloaded by yt_dlp.")
    # Return the largest file (likely merged video)
    files_sorted = sorted(files, key=lambda p: p.stat().st_size, reverse=True)
    return str(files_sorted[0])

@dp.message_handler()
async def handle_download(message: types.Message):
    url = (message.text or "").strip()
    if not url or "http" not in url:
        await message.reply("❗ أرسل رابط صحيح (يبدأ بـ http/https).")
        return

    sent_msg = await message.answer("⏳ جاري المعالجة... انتظر ثواني")

    # Acquire semaphore to limit concurrency
    async with DOWNLOAD_SEMAPHORE:
        tmpdir = tempfile.mkdtemp(prefix="yt_dl_")
        loop = asyncio.get_event_loop()
        downloaded_path = None

        try:
            # Run blocking download in threadpool
            downloaded_path = await loop.run_in_executor(None, run_ydl_sync, url, tmpdir)
            file_size = os.path.getsize(downloaded_path)
            logger.info("Downloaded file: %s (%d bytes)", downloaded_path, file_size)

            # If file is too large for bot API (adjust policy), provide alternative
            if file_size > TG_SAFE_SIZE_BYTES:
                await message.reply(
                    "✅ تم تحميل الملف لكن الحجم كبير جدًا للإرسال عبر البوت.\n"
                    "يمكنك تنزيله من الخادم لاحقًا أو استخدم رفعًا خارجيًا (S3/Dropbox) — اطلُب مني أن أضيف رفعًا تلقائيًا."
                )
                # Optionally: upload to cloud here and send link
            else:
                # Send as video (or document if you prefer)
                with open(downloaded_path, "rb") as f:
                    await bot.send_video(chat_id=message.chat.id, video=f, caption="✅ تم التحميل بواسطة MB")

            await bot.delete_message(chat_id=message.chat.id, message_id=sent_msg.message_id)

        except Exception as e:
            logger.exception("Download or send failed")
            await message.reply("❌ عذراً، هذا الرابط غير مدعوم أو حصل خطأ. حاول مرة ثانية لاحقًا.")
        finally:
            # Cleanup
            try:
                shutil.rmtree(tmpdir)
            except Exception:
                logger.warning("Failed to remove temp dir %s", tmpdir)

if __name__ == '__main__':
    logger.info("البوت شغال دلوقتي..")
    executor.start_polling(dp, skip_updates=True)
```

اقتراحات إضافية (اختيارية)
- استخدم storage/DB لحفظ سجل التحميلات، والحد من إساءة الاستخدام.
- أضف جدول أسعار أو نظام مدفوعات/قـصّر التحميلات اليومية للمستخدمين.
- لو تريد إرسال فيديوهات كبيرة إلى المستخدمين، أنفذ رفعًا إلى S3/Cloudflare R2 ثم أرسل رابطًا (أو استخدم Telegram file API الكبير عبر getFile + upload إضافي إذا متاح).
- أضف progress hook من yt_dlp لإرسال تحديثات تقدم التحميل للمستخدم.
- ضع البوت في Docker وابدأ systemd أو process manager، واستخدم secrets للـ token.

هل أعدل الكود لشكل آخر (مثلاً: رفع تلقائي إلى S3 بعد التحميل وإرسال رابط، أو دعم التحميل بدون علامة مائية لـ TikTok إذا متاح)؟

*System: User retried message*
