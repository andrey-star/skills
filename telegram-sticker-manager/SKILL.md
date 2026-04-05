---
name: telegram-sticker-manager
description: Manage the 'Andrey's Pack' Telegram sticker set. Use to add or modify stickers from images, GIFs, or videos. Handles all Telegram formatting constraints automatically.
---

# [Andrey's Pack](https://t.me/addstickers/andreys_pack_by_fastrazor_bot) Telegram Sticker Management Skill

This skill allows an agent to programmatically manage [Andrey's Pack](https://t.me/addstickers/andreys_pack_by_fastrazor_bot) using the Telegram Bot API.

## Core Capabilities
- **Validation:** Check if media files meet Telegram's strict technical requirements.
- **Transformation:** Automatically convert GIFs, MP4s, or images into compliant VP9 `.webm` files.
- **Pack Management:** Create new packs or add/replace stickers in existing packs.

## Technical Constraints for Video Stickers (.webm)
All video stickers MUST adhere to these limits:
- **Format:** WebM (VP9 codec).
- **Dimensions:** Exactly 512x512 pixels (at least one side must be 512, the other 512 or less).
- **Duration:** Maximum 3.0 seconds.
- **File Size:** Maximum 256 KB.
- **Audio:** Must NOT have an audio stream.
- **Frame Rate:** Maximum 30 FPS.

## Required Environment & Context
- `TELEGRAM_BOT_TOKEN`: Must be available in the environment.
- `USER_ID`: `260367801` (Andrey).
- `PACK_NAME`: `andreys_pack_by_fastrazor_bot`.
- **Tools:** `ffmpeg`, `ffprobe`, `curl`, `jq`. (Ensure these are installed and in the PATH).

## Workflow for Adding a New Sticker

### 1. Research & Analysis
- **Analyze Source:** Identify the source file (GIF, MP4, MOV) and run `ffprobe` to get dimensions, duration, and frame rate.
- **Guardrail Check:** Compare the results against Telegram's limits (3s, 512px, 256KB, no audio, 30fps).
- **Report & Confirm:** If the file does NOT match, the agent MUST highlight the conflicts and ask for your preference.
  - *Example:* "File is 10s long. Should I **(a)** Speed it up to 3s, or **(b)** Crop to the first 3s?"
  - *Example:* "File is 1920x1080. Should I **(a)** Crop to a 512x512 square, or **(b)** Scale to a 512x288 rectangle?"

### 2. Transformation (FFmpeg)
Use the following command pattern to ensure compliance:
```bash
ffmpeg -i input_file -vf "setpts=(min(3,d)/d)*PTS,scale=512:512:force_original_aspect_ratio=increase,crop=512:512" -c:v libvpx-vp9 -b:v 500k -an -t 3 -pix_fmt yuva420p output.webm
```

#### Command Breakdown:
| Parameter | Purpose | How to Customize |
| :--- | :--- | :--- |
| `-vf "setpts=..."` | **Speed:** Forces the entire clip into 3 seconds. | Remove this if you'd rather crop time (see below). |
| `scale=512:512...` | **Size:** Scales to cover 512x512 without stretching. | Change `512:512` to `512:-1` for non-square (max 512). |
| `crop=512:512` | **Square:** Crops the video to a perfect square. | Remove this if you want a rectangular sticker. |
| `-c:v libvpx-vp9` | **Codec:** The only video format Telegram accepts. | **Do not change.** |
| `-b:v 500k` | **Quality:** Keeps file size under 256KB. | Increase to `600k` for high detail, decrease if file is >256KB. |
| `-an` | **Audio:** Removes all sound streams. | **Do not change** (required). |
| `-pix_fmt yuva420p`| **Alpha:** Keeps transparency from GIFs/PNGs. | **Do not change.** |

#### Customization Examples:
- **To take a specific 3-second segment (Time Crop):**
  Replace `setpts` with `-ss 00:00:05 -t 3` (starts at 5 seconds, takes 3).
- **To keep original aspect ratio (Rectangular):**
  Use `-vf "scale=512:-1"` (width is 512, height scales automatically).
- **To make it exactly 3 seconds (without speeding up):**
  Use `-t 3` at the end to truncate the clip.

### 3. Emoji Selection
- **Analyze Media:** Use `read_file` to view the image/video.
- **Suggest Emojis:** Based on the visual content, suggest 3-5 relevant emojis to the user.
- **User Confirmation:** Wait for the user to either pick from your suggestions or provide their own. **Do NOT proceed without a confirmed emoji.**

### 4. Execution (API Call)
Once the emoji and the transformed file are ready, add the sticker to the pack:
```bash
curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/addStickerToSet" \
     -F "user_id=260367801" \
     -F "name=andreys_pack_by_fastrazor_bot" \
     -F "sticker={\"sticker\":\"attach://sticker_file\", \"emoji_list\":[\"EMOJI_HERE\"], \"format\":\"video\"}" \
     -F "sticker_file=@/path/to/sticker.webm"
```

## Validation Checklist
- [ ] Is duration ≤ 3.0s?
- [ ] Is dimensions exactly 512x512?
- [ ] Is file size < 256KB?
- [ ] Is the codec VP9?
- [ ] Is there no audio?
