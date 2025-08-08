# 🎓 Chaptly (YouTube Tutor Chatbot)

**A conversational learning assistant powered by YouTube transcripts + vector search + LLMs.**  
Users paste a YouTube video link and ask questions or get guided to the relevant section — like an AI-powered bookmark navigator.


> **Note:** This is a public version of the repository with limited features. Please reach out privately to the repository owner for access to the full private repository with complete functionality.

---

## 🚀 Purpose

This app helps learners:
- Extract and understand **key segments** of long educational videos.
- Ask topic-based questions about a video (e.g., "Where does it explain overfitting?").
- Navigate content via **smart bookmarks**, labeled with section titles and timestamps.

    1.	✅ Allow only verified channels by default
    2.	⚠️ Show warnings when users submit other content:
"This channel is not verified for educational reuse. Please confirm you have permission or that the content falls under fair use."

This is ideal for:
- Students revising lectures (e.g., Coursera, Khan Academy).
- Lifelong learners who want to save time.
- Tutors creating quizzes from video lectures.

---

## 🧱 Architecture Overview

```
                        ┌─────────────────────┐
                        │  User Frontend (UI) │
                        └────────┬────────────┘
                                 │
                ┌─────────────────────────────┐
                │  Flask / FastAPI Backend    │
                └─────────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
 ┌──────────────┐       ┌─────────────────┐     ┌─────────────────────┐
 │ Transcript   │       │  Vector Store   │     │    LLM (OpenAI)     │
 │ Fetcher      │──────▶│ (FAISS/Chroma)  │◀────│  via RAG pipeline   │
 └──────────────┘       └─────────────────┘     └─────────────────────┘
                                 │
                       ┌─────────▼─────────┐
                       │  Bookmark Annotator│
                       └────────────────────┘
```

---

## 🔧 Tech Stack

| Layer          | Tools / Libraries                           |
|----------------|---------------------------------------------|
| Frontend       | Streamlit / React                           |
| Backend        | FastAPI or Flask                            |
| Transcript     | `youtube_transcript_api`, Whisper (fall back)|
| Embedding      | OpenAI `text-embedding-3-small`             |
| Vector Store   | FAISS or ChromaDB                           |
| LLM            | OpenAI GPT-4 (RAG-style QA + Summarization) |
| Bookmarking    | Clustering or GPT-based Topic Segmentation  |

---

## 🛠 Features

- ✅ Paste YouTube link → auto transcript extraction
- ✅ Ask questions in chat → get answer + timestamp
- ✅ Auto-generated section bookmarks with summaries
- ✅ Clickable navigation using YouTube player

---

## ⚠️ Legal & Compliance

- Only works with **public videos** that have transcript enabled.
- Does not download or host video/audio content.
- Pre-approved YouTube channels include:

  **Academic Institutions:**
  - MIT OpenCourseWare, Stanford Online, Harvard University, Yale Courses
  
  **Educational Platforms:**
  - Khan Academy, Coursera, edX, Udacity
  
  **Educational Content Creators:**
  - Crash Course, TED-Ed, 3Blue1Brown, Numberphile, Computerphile, SciShow, 
    Veritasium, MinutePhysics, CodeWithMosh, freeCodeCamp.org

- Non-approved channels require user confirmation of appropriate permissions.
- See [Terms of Use](./terms-of-use.md) for full policy and complete channel list.

---

## 🧪 Setup & Development

### System Requirements

#### FFmpeg Installation (Required for Whisper Transcription)

**macOS:**
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add Homebrew to PATH (for Apple Silicon Macs)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# Install ffmpeg
brew install ffmpeg

# Verify installation
ffmpeg -version
```

**Windows:**
1. Download FFmpeg from the official website: https://ffmpeg.org/download.html#build-windows
2. Extract the ZIP file to a folder (e.g., `C:\ffmpeg`)
3. Add FFmpeg to your PATH:
   - Right-click on "This PC" or "My Computer" and select "Properties"
   - Click on "Advanced system settings"
   - Click on "Environment Variables"
   - Under "System variables", select "Path" and click "Edit"
   - Click "New" and add the path to the `bin` folder (e.g., `C:\ffmpeg\bin`)
   - Click "OK" on all dialogs to save
4. Verify installation by opening a new Command Prompt and typing:
   ```
   ffmpeg -version
   ```

**Linux:**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ffmpeg

# CentOS/RHEL
sudo yum install ffmpeg

# Verify installation
ffmpeg -version
```

### Python Dependencies

```bash
# Install dependencies
pip install -r requirements.txt

# Set your environment variables
export OPENAI_API_KEY=sk-...
export VECTOR_DB=chroma  # or 'faiss'

# Run the backend server
python app.py
```

---

## 📍 Roadmap

- [ ] Add multilingual transcript support
- [ ] "Quiz me" mode (generate questions)
- [ ] Save personal bookmarks per user
- [ ] Host on Hugging Face or Streamlit Cloud

---

## 👥 Contributing

Pull requests welcome! Please file issues for bugs or feature suggestions.

---

## 📄 License

General Public License — see [LICENSE](./LICENSE)
