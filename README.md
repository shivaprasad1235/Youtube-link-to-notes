# Youtube-link-to-notes
THis application helps you to takes notes from an you tube vedio
from flask import Flask, request, render_template, send_file
import yt_dlp
import speech_recognition as sr
from transformers import pipeline
from reportlab.lib.pagesizes import letter
from reportlab.platypus import SimpleDocTemplate, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
import os

app = Flask(__name__)

# Suppress oneDNN message
os.environ["TF_ENABLE_ONEDNN_OPTS"] = "0"

# Load summarization model once (outside routes for efficiency)
try:
    print("Loading the summarization model...")
    summarizer = pipeline("summarization", model="t5-small")  # Switched to t5-small for better performance
    print("Model loaded successfully!")
except KeyboardInterrupt:
    print("\nModel loading interrupted by user. Exiting...")
    exit(1)
except Exception as e:
    print(f"Error loading model: {e}")
    exit(1)

# Home page
@app.route('/')
def index():
    return render_template('index.html', error=None)

# Process YouTube link and generate PDF notes
@app.route('/generate', methods=['POST'])
def generate_notes():
    youtube_url = request.form['youtube_url']

    try:
        # Step 1: Download audio from YouTube
        ydl_opts = {
            'format': 'bestaudio',
            'outtmpl': 'audio.%(ext)s',
            'postprocessors': [{
                'key': 'FFmpegExtractAudio',
                'preferredcodec': 'wav',
                'preferredquality': '192',
            }],
        }
        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.download([youtube_url])

        # Step 2: Transcribe audio
        recognizer = sr.Recognizer()
        audio_file = "audio.wav"
        with sr.AudioFile(audio_file) as source:
            audio_data = recognizer.record(source)
            transcript = recognizer.recognize_google(audio_data)

        # Step 3: Summarize transcript
        summary = summarizer(transcript, max_length=130, min_length=30, do_sample=False)[0]['summary_text']

        # Step 4: Create PDF
        pdf_file = "notes.pdf"
        doc = SimpleDocTemplate(pdf_file, pagesize=letter)
        styles = getSampleStyleSheet()
        story = [Paragraph(f"<b>Video Notes</b><br/><br/>{summary}", styles["Normal"])]
        doc.build(story)

        # Step 5: Clean up and send file
        os.remove(audio_file)
        return send_file(pdf_file, as_attachment=True, download_name='video_notes.pdf')

    except Exception as e:
        # Clean up audio file if it exists
        if os.path.exists("audio.wav"):
            os.remove("audio.wav")
        return render_template('index.html', error=f"Error: {str(e)}")

if __name__ == '__main__':
    app.run(debug=True)
