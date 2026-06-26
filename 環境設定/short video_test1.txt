import os

# ==========================================
# 0. 針對 Linux / Streamlit Cloud 的環境修正
# ==========================================
# 檢查是否在 Linux 環境，並強制指定 ImageMagick 的二進位檔案路徑
if os.name != 'nt':  # 'nt' 代表 Windows，如果不是 Windows 就是 Linux/Mac
    os.environ["IMAGEMAGICK_BINARY"] = "/usr/bin/convert"
    
    # 某些新版 Linux 的 ImageMagick 限制了安全策略，會禁止 TextClip 渲染文字
    # 以下指令可以解除 ImageMagick 對於 PDF/TEXT 的讀寫限制
    os.system("sed -i 's/rights=/"none/" pattern=/"PDF/"/rights=/"read|write/" pattern=/"PDF/"/g' /etc/ImageMagick-6/policy.xml 2>/dev/null")



import streamlit as st
import random
import tempfile
from moviepy.editor import (
    ImageClip, VideoFileClip, concatenate_videoclips, 
    AudioFileClip, CompositeVideoClip, CompositeAudioClip, TextClip
)
import moviepy.video.fx.all as vfx

# ==========================================
# 1. 核心設定：產業與風格模板資料庫
# ==========================================
STYLE_TEMPLATES = {
    "cinematic": {
        "name": "微電影工作室（沉穩高質感）",
        "filter": "cinematic",
        "hook_effect": "fade_slow",
        "text_font": "Microsoft-JhengHei-Bold",
        "text_color": "#FFFFFF",
        "stroke_color": "#000000",
        "stroke_width": 2,
        "text_pos": ("center", "bottom"),
        "sfx_pattern": {0.0: "ambient_swell.mp3", 2.0: "sub_boom.mp3"}
    },
    "viral_vlog": {
        "name": "極速網紅自媒體（快節奏重口味）",
        "filter": "cyberpunk",
        "hook_effect": "zoom_fast",
        "text_font": "Microsoft-JhengHei-Bold",
        "text_color": "#FFFF00",
        "stroke_color": "#000000",
        "stroke_width": 5,
        "text_pos": ("center", "center"),
        "sfx_pattern": {0.0: "sfx_whoosh.mp3", 1.5: "sfx_glitch.mp3"}
    },
    "japanese_fresh": {
        "name": "日系清新小資（高亮通透）",
        "filter": "bright_soft",
        "hook_effect": "flash_white",
        "text_font": "Microsoft-JhengHei",
        "text_color": "#333333",
        "stroke_color": "#FFFFFF",
        "stroke_width": 3,
        "text_pos": ("center", 300),
        "sfx_pattern": {0.0: "sfx_ding.mp3", 3.0: "sfx_camera.mp3"}
    }
}

# ==========================================
# 2. 核心引擎：特效與調色函式
# ==========================================
def apply_style_filter(clip, filter_type):
    if filter_type == "cinematic":
        return clip.fx(vfx.lum_contrast, contrast=0.25, lum=-0.05).fx(vfx.colorx, 0.9)
    elif filter_type == "cyberpunk":
        return clip.fx(vfx.lum_contrast, contrast=0.35).fx(vfx.colorx, 1.3)
    elif filter_type == "bright_soft":
        return clip.fx(vfx.lum_contrast, contrast=-0.1, lum=0.1).fx(vfx.colorx, 0.85)
    return clip

def apply_visual_hook(clip, effect_type, duration=1.0):
    if effect_type == "zoom_fast":
        return clip.fx(vfx.resize, lambda t: 1.0 + 0.4 * (t/duration) if t < duration else 1.4)
    elif effect_type == "flash_white":
        return clip.fx(vfx.fadein, 0.3)
    elif effect_type == "fade_slow":
        return clip.fx(vfx.fadein, 0.8)
    return clip

def generate_saas_video(user_media_paths, style_id, slogan_text, shop_name, logo_path, output_path):
    """ 後端剪輯核心引擎 """
    target_size = (1080, 1920)
    cfg = STYLE_TEMPLATES[style_id]
    
    # 處理與串接素材
    clips = []
    for media in user_media_paths[:5]:
        if media.endswith(('.mp4', '.mov')):
            clip = VideoFileClip(media).subclip(0, 3).resize(target_size)
        else:
            clip = ImageClip(media).set_duration(3).resize(target_size)
        clips.append(clip)
    
    final_media_clip = concatenate_videoclips(clips, method="compose")
    final_media_clip = apply_style_filter(final_media_clip, cfg["filter"])
    final_media_clip = apply_visual_hook(final_media_clip, cfg["hook_effect"])

    # 動態字幕
    if slogan_text:
        title_clip = TextClip
            slogan_text, fontsize=60, color=cfg["text_color"], font="./fonts/NotoSansTC-Black.ttf",
            stroke_color=cfg["stroke_color"], stroke_width=cfg["stroke_width"],
            size=(900, None), method='caption'
        ).set_duration(2.5).set_position(cfg["text_pos"])
        fade_time = 0.1 if style_id == "viral_vlog" else 0.5
        title_clip = title_clip.fx(vfx.fadein, fade_time)
        final_video = CompositeVideoClip([final_media_clip, title_clip])
    else:
        final_video = final_media_clip

    # 右上角品牌浮水印
    if logo_path and os.path.exists(logo_path):
        watermark = ImageClip(logo_path).set_duration(final_video.duration).resize(width=160)
        watermark = watermark.set_position((840, 50)).set_opacity(0.8)
        final_video = CompositeVideoClip([final_video, watermark])
    elif shop_name:
        name_clip = TextClip(f"@{shop_name}", fontsize=28, color="white", font=cfg["text_font"]).set_duration(final_video.duration)
        name_clip = name_clip.set_position((800, 60)).set_opacity(0.6)
        final_video = CompositeVideoClip([final_video, name_clip])

    video_duration = final_video.duration
    audio_clips = []
    
    # ==========================================
    # 音樂庫隨機抽選 (讀取我們剛建好的 bgm 資料夾)
    # ==========================================
    bgm_dir = f"./bgm/{style_id}"
    
    # 確保資料夾存在
    if os.path.exists(bgm_dir):
        # 嚴格過濾，只抓取 .mp3 結尾的檔案
        songs = [os.path.join(bgm_dir, f) for f in os.listdir(bgm_dir) if f.endswith('.mp3')]
        
        # 確保資料夾裡面真的有 mp3 檔案才執行
        if songs:
            import random
            selected_bgm = random.choice(songs)
            bgm_volume = 0.2 if style_id == "viral_vlog" else 0.3
            
            # 使用 audio_loop：音樂不夠長會自動重播補滿，太長會自動截斷
            from moviepy.audio.fx.all import audio_loop
            bgm_audio = AudioFileClip(selected_bgm)
            bgm_audio = audio_loop(bgm_audio, duration=video_duration).volumex(bgm_volume)
            audio_clips.append(bgm_audio)
    
    # ==========================================
    # 載入特效音 (保持不變)
    # ==========================================
    for trigger_time, sfx_name in cfg["sfx_pattern"].items():
        sfx_path = f"./assets/{sfx_name}"
        if os.path.exists(sfx_path) and trigger_time < video_duration:
            sfx_audio = AudioFileClip(sfx_path).set_start(trigger_time).volumex(1.0)
            audio_clips.append(sfx_audio)

    # 合併所有音軌到影片中
    if audio_clips:
        final_video = final_video.set_audio(CompositeAudioClip(audio_clips))

    # 確保影片總長不超過 60 秒，避免伺服器超載
    if final_video.duration > 60:
        final_video = final_video.subclip(0, 60)

    # ==========================================
    # 執行渲染輸出 (保持不變)
    # ==========================================
    final_video.write_videofile(
        output_path, fps=24, codec="libx264", audio_codec="aac", threads=2
    )

# ==========================================
# 3. Streamlit 前端網頁介面
# ==========================================
st.set_page_config(page_title="SaaS 短影音自動生成器", layout="centered")
st.title("🎬 SaaS 短影音自動化行銷平台")
st.subheader("影音業專專屬測試版 ── 一鍵生成你的爆款短影音")

# 介面輸入區（保持不變）
style_choice = st.selectbox(
    "步驟 1：選擇視聽風格模板", 
    ["微電影工作室（沉穩高質感）", "極速網紅自媒體（快節奏重口味）", "日系清新小資（高亮通透）"]
)
style_id_map = {
    "微電影工作室（沉穩高質感）": "cinematic",
    "極速網紅自媒體（快節奏重口味）": "viral_vlog",
    "日系清新小資（高亮通透）": "japanese_fresh"
}
selected_style_id = style_id_map[style_choice]

uploaded_files = st.file_uploader(
    "步驟 2：上傳素材影片或照片（可多選，最多5個）", 
    accept_multiple_files=True, 
    type=["jpg", "jpeg", "png", "mp4", "mov"]
)

slogan = st.text_input("步驟 3：輸入爆款標語", "🔥 本月最扯下殺！")
shop_name = st.text_input("步驟 4：輸入您的品牌/店名", "光影影像工作室")
logo_file = st.file_uploader("步驟 5：上傳品牌 LOGO (透明底 PNG 佳，選填)", type=["png"])

# 按鈕執行
if st.button("🚀 一鍵生成爆款短影音"):
    if not uploaded_files:
        st.warning("⚠️ 請至少上傳一個媒體素材（照片或影片）才能開始剪輯喔！")
    else:
        st.info("⏳ 影片排隊處理中... 雲端環境渲染大約需要 1~2 分鐘，請勿關閉網頁...")
        
        # 建立臨時目錄來儲存消費者上傳的檔案
        with tempfile.TemporaryDirectory() as temp_dir:
            user_media_paths = []
            
            # 保存上傳的素材
            for f in uploaded_files:
                temp_path = os.path.join(temp_dir, f.name)
                with open(temp_path, "wb") as buffer:
                    buffer.write(f.read())
                user_media_paths.append(temp_path)
            
            # 保存上傳的 LOGO
            logo_path = None
            if logo_file:
                logo_path = os.path.join(temp_dir, "user_logo.png")
                with open(logo_path, "wb") as buffer:
                    buffer.write(logo_file.read())
            
            # 設定輸出路徑
            output_video_path = os.path.join(temp_dir, "final_output.mp4")
            
            try:
                # 呼叫剪輯核心
                generate_saas_video(
                    user_media_paths=user_media_paths,
                    style_id=selected_style_id,
                    slogan_text=slogan,
                    shop_name=shop_name,
                    logo_path=logo_path,
                    output_path=output_video_path
                )
                
                # 渲染成功，將影片讀入網頁記憶體
                st.success("🎉 短影音生成成功！請在下方預覽與下載：")
                with open(output_video_path, "rb") as video_file:
                    video_bytes = video_file.read()
                    
                    # 呈現畫面與下載按鈕
                    st.video(video_bytes)
                    st.download_button(
                        label="💾 下載生成的短影音",
                        data=video_bytes,
                        file_name="viral_short_video.mp4",
                        mime="video/mp4"
                    )
            except Exception as e:
                st.error(f"❌ 渲染過程中發生錯誤：{e}")
                st.info("提示：如果遇到中文字型問題，請確保雲端環境已正確載入中文字型。")
            
            # === 【關鍵優化】在這裡手動觸發 Python 垃圾回收機制釋放記憶體 ===
            finally:
                import gc
                gc.collect()  # 強制清理剛才解散的臨時檔案與視訊殘留記憶體