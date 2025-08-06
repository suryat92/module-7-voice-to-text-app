from flask import Flask, request, render_template, jsonify, redirect, url_for
import speech_recognition as sr
import os
import sqlite3
from datetime import datetime
from dotenv import load_dotenv
import tempfile
from werkzeug.utils import secure_filename

load_dotenv()

app = Flask(__name__)

# Configuration
app.config['UPLOAD_FOLDER'] = os.getenv('UPLOAD_FOLDER', 'uploads')
app.config['MAX_CONTENT_LENGTH'] = int(os.getenv('MAX_CONTENT_LENGTH', 16777216))
DATABASE_PATH = os.getenv('DATABASE_PATH', 'transcriptions.db')

# Ensure upload directory exists
os.makedirs(app.config['UPLOAD_FOLDER'], exist_ok=True)

# Initialize speech recognizer
recognizer = sr.Recognizer()

def init_db():
    """Initialize the SQLite database"""
    with sqlite3.connect(DATABASE_PATH) as conn:
        conn.execute('''
            CREATE TABLE IF NOT EXISTS transcriptions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                filename TEXT NOT NULL,
                original_text TEXT,
                confidence REAL,
                processing_time REAL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        conn.commit()

def allowed_file(filename):
    """Check if file extension is allowed"""
    ALLOWED_EXTENSIONS = {'wav', 'mp3', 'flac', 'aiff', 'aac'}
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def transcribe_audio(file_path):
    """Convert audio file to text using speech recognition"""
    try:
        start_time = datetime.now()
        
        with sr.AudioFile(file_path) as source:
            # Adjust for ambient noise
            recognizer.adjust_for_ambient_noise(source)
            # Record the audio
            audio_data = recognizer.record(source)
        
        # Recognize speech using Google Speech Recognition
        try:
            text = recognizer.recognize_google(audio_data, show_all=False)
            processing_time = (datetime.now() - start_time).total_seconds()
            return {
                'success': True,
                'text': text,
                'confidence': 0.95,  # Google API doesn't return confidence
                'processing_time': processing_time
            }
        except sr.UnknownValueError:
            return {
                'success': False,
                'error': 'Could not understand audio',
                'processing_time': (datetime.now() - start_time).total_seconds()
            }
        except sr.RequestError as e:
            return {
                'success': False,
                'error': f'Could not request results; {e}',
                'processing_time': (datetime.now() - start_time).total_seconds()
            }
    
    except Exception as e:
        return {
            'success': False,
            'error': f'Error processing audio: {str(e)}',
            'processing_time': 0
        }

@app.route('/')
def index():
    """Render the main page"""
    return render_template('index.html')

@app.route('/upload', methods=['POST'])
def upload_file():
    """Handle file upload and transcription"""
    if 'audio_file' not in request.files:
        return jsonify({'error': 'No file selected'}), 400
    
    file = request.files['audio_file']
    if file.filename == '':
        return jsonify({'error': 'No file selected'}), 400
    
    if not allowed_file(file.filename):
        return jsonify({'error': 'File type not supported. Please upload WAV, MP3, FLAC, AIFF, or AAC files'}), 400
    
    try:
        # Save uploaded file
        filename = secure_filename(file.filename)
        file_path = os.path.join(app.config['UPLOAD_FOLDER'], filename)
        file.save(file_path)
        
        # Transcribe audio
        result = transcribe_audio(file_path)
        
        # Save to database
        with sqlite3.connect(DATABASE_PATH) as conn:
            if result['success']:
                conn.execute('''
                    INSERT INTO transcriptions (filename, original_text, confidence, processing_time)
                    VALUES (?, ?, ?, ?)
                ''', (filename, result['text'], result['confidence'], result['processing_time']))
            else:
                conn.execute('''
                    INSERT INTO transcriptions (filename, original_text, processing_time)
                    VALUES (?, ?, ?)
                ''', (filename, result.get('error', 'Transcription failed'), result['processing_time']))
            conn.commit()
        
        # Clean up uploaded file
        os.remove(file_path)
        
        if result['success']:
            return jsonify({
                'success': True,
                'text': result['text'],
                'confidence': result['confidence'],
                'processing_time': result['processing_time'],
                'filename': filename
            })
        else:
            return jsonify({
                'success': False,
                'error': result['error'],
                'processing_time': result['processing_time']
            }), 400
            
    except Exception as e:
        return jsonify({'error': f'Server error: {str(e)}'}), 500

@app.route('/history')
def get_history():
    """Get transcription history"""
    try:
        with sqlite3.connect(DATABASE_PATH) as conn:
            conn.row_factory = sqlite3.Row
            cursor = conn.execute('''
                SELECT id, filename, original_text, confidence, processing_time, created_at
                FROM transcriptions
                ORDER BY created_at DESC
                LIMIT 50
            ''')
            history = [dict(row) for row in cursor.fetchall()]
        
        return jsonify({'history': history})
    
    except Exception as e:
        return jsonify({'error': f'Database error: {str(e)}'}), 500

@app.route('/stats')
def get_stats():
    """Get application statistics"""
    try:
        with sqlite3.connect(DATABASE_PATH) as conn:
            cursor = conn.execute('''
                SELECT 
                    COUNT(*) as total_transcriptions,
                    AVG(processing_time) as avg_processing_time,
                    COUNT(CASE WHEN confidence > 0 THEN 1 END) as successful_transcriptions
                FROM transcriptions
            ''')
            stats = dict(cursor.fetchone())
        
        return jsonify(stats)
    
    except Exception as e:
        return jsonify({'error': f'Database error: {str(e)}'}), 500

@app.route('/health')
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'service': 'voice-to-text-app',
        'timestamp': datetime.now().isoformat()
    })

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=True)