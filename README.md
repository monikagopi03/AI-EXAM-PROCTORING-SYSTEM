# AI-EXAM-PROCTORING-SYSTEM
from flask import Flask, render_template, request, redirect, url_for, session, flash
import sqlite3
import cv2
import face_recognition
import numpy as np
import pickle
import os

app = Flask(__name__)
app.secret_key = 'your_secret_key'
DB_PATH = "database.db"
UPLOAD_FOLDER = 'uploads'
os.makedirs(UPLOAD_FOLDER, exist_ok=True)

# Initialize database
def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # Create users table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL
    )
    """)
    
    # Create students table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS students (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        warnings INTEGER DEFAULT 0,
        malpractice_records INTEGER DEFAULT 0
    )
    """)
    
    # Create face encodings table
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS face_encodings (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        student_id INTEGER,
        encoding BLOB,
        FOREIGN KEY(student_id) REFERENCES students(id)
    )
    """)
    
    conn.commit()
    conn.close()

# Route: Home
@app.route('/')
def index():
    return render_template('index.html')

# Route: Registration
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        try:
            cursor.execute("INSERT INTO users (username, password) VALUES (?, ?)", (username, password))
            conn.commit()
            flash('Registration successful! Please log in.', 'success')
            return redirect(url_for('login'))
        except sqlite3.IntegrityError:
            flash('Username already exists. Please choose another.', 'danger')
        finally:
            conn.close()
    return render_template('register.html')

# Route: Login
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        if username == "admin" and password == "1234":
            session['username'] = username
            return redirect(url_for('admin_dashboard'))
        else:
            conn = sqlite3.connect(DB_PATH)
            cursor = conn.cursor()
            cursor.execute("SELECT * FROM users WHERE username=? AND password=?", (username, password))
            user = cursor.fetchone()
            conn.close()
            if user:
                session['username'] = username
                return redirect(url_for('user_dashboard'))
            else:
                flash('Invalid username or password. Please try again.', 'danger')
    return render_template('login.html')

# Route: Admin Dashboard
@app.route('/admin_dashboard')
def admin_dashboard():
    if 'username' in session and session['username'] == 'admin':
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM students")
        students = cursor.fetchall()
        conn.close()
        return render_template('admin_dashboard.html', students=students)
    return redirect(url_for('login'))

# Route: User Dashboard
@app.route('/user_dashboard')
def user_dashboard():
    if 'username' in session:
        return render_template('user_dashboard.html', username=session['username'])
    return redirect(url_for('login'))

# Route: Logout
@app.route('/logout')
def logout():
    session.pop('username', None)
    flash('You have been logged out.', 'success')
    return redirect(url_for('login'))

# Route: Register Student with Face
@app.route('/register_student', methods=['GET', 'POST'])
def register_student():
    if 'username' in session and session['username'] == 'admin':
        if request.method == 'POST':
            name = request.form['name']
            file = request.files['image']
            if file:
                image_path = os.path.join(UPLOAD_FOLDER, file.filename)
                file.save(image_path)

                # Encode face
                image = face_recognition.load_image_file(image_path)
                face_encodings = face_recognition.face_encodings(image)
                if face_encodings:
                    face_encoding = face_encodings[0]

                    conn = sqlite3.connect(DB_PATH)
                    cursor = conn.cursor()
                    cursor.execute("INSERT INTO students (name, warnings, malpractice_records) VALUES (?, 0, 0)", (name,))
                    student_id = cursor.lastrowid

                    # Save face encoding
                    cursor.execute("INSERT INTO face_encodings (student_id, encoding) VALUES (?, ?)", 
                                   (student_id, pickle.dumps(face_encoding)))
                    conn.commit()
                    conn.close()

                    flash(f"Student {name} registered successfully!", 'success')
                else:
                    flash("No face detected in the image. Please try again.", 'danger')
            return redirect(url_for('register_student'))
        return render_template('register_student.html')
    return redirect(url_for('login'))

@app.route('/monitor')
def monitor():
    if 'username' in session and session['username'] == 'admin':
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("SELECT student_id, encoding FROM face_encodings")
        stored_encodings = cursor.fetchall()
        conn.close()

        known_face_encodings = []
        known_face_ids = []
        for student_id, encoding in stored_encodings:
            known_face_encodings.append(pickle.loads(encoding))
            known_face_ids.append(student_id)

        video_capture = cv2.VideoCapture(0)
        while True:
            ret, frame = video_capture.read()
            rgb_frame = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            face_locations = face_recognition.face_locations(rgb_frame)
            face_encodings = face_recognition.face_encodings(rgb_frame, face_locations)

            for face_encoding, face_location in zip(face_encodings, face_locations):
                matches = face_recognition.compare_faces(known_face_encodings, face_encoding)
                face_distances = face_recognition.face_distance(known_face_encodings, face_encoding)
                best_match_index = np.argmin(face_distances)

                if matches[best_match_index]:
                    student_id = known_face_ids[best_match_index]
                    conn = sqlite3.connect(DB_PATH)
                    cursor = conn.cursor()
                    cursor.execute("SELECT name, warnings, malpractice_records FROM students WHERE id=?", (student_id,))
                    student = cursor.fetchone()

                    if student:
                        student_name, warnings, malpractice_records = student

                        # Check if student is looking away or showing unwanted behavior
                        top, right, bottom, left = face_location
                        # Let's assume if the face location is too far to the left/right, it's considered looking away
                        if left < 50 or right > frame.shape[1] - 50:  # Adjust based on your webcam resolution
                            warnings += 1

                        if warnings >= 3 and malpractice_records < 3:
                            malpractice_records += 1
                            flash(f"Student {student_name} reached malpractice limit.", 'danger')

                        # Update the student records in the database
                        cursor.execute("""
                            UPDATE students
                            SET warnings = ?, malpractice_records = ?
                            WHERE id = ?
                        """, (warnings, malpractice_records, student_id))

                        conn.commit()

                    conn.close()

                    # Draw rectangle and name around the face
                    top, right, bottom, left = face_location
                    cv2.rectangle(frame, (left, top), (right, bottom), (0, 255, 0), 2)
                    cv2.putText(frame, student_name, (left, top - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, (255, 255, 255), 2)

            cv2.imshow("Monitor", frame)

            # Check if user wants to stop the webcam feed (press 'q' to exit)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                break

        video_capture.release()
        cv2.destroyAllWindows()
        return redirect(url_for('admin_dashboard'))
    
    return redirect(url_for('login'))



@app.route('/update_student/<int:student_id>', methods=['POST'])
def update_student(student_id):
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()

    # Fetch student data
    cursor.execute("SELECT warnings, malpractice_records FROM students WHERE id=?", (student_id,))
    student = cursor.fetchone()

    if student:
        warnings = student[0]
        malpractice_records = student[1]

        # Logic to increase warnings or malpractice records
        if warnings < 3:
            warnings += 1
        elif warnings >= 3 and malpractice_records < 3:
            malpractice_records += 1
        elif malpractice_records >= 3:
            flash('Student has reached the maximum malpractice limit.', 'danger')

        # Update the student record in the database
        cursor.execute("""
            UPDATE students
            SET warnings = ?, malpractice_records = ?
            WHERE id = ?
        """, (warnings, malpractice_records, student_id))

        conn.commit()
        conn.close()

    return redirect(url_for('admin_dashboard'))


if __name__ == "__main__":
    init_db()
    app.run(debug=True)
