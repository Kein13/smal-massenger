import os
import logging
import webbrowser
import random
import string
import secrets
import sqlite3
from datetime import datetime
from flask import Flask, render_template, request, session, redirect, url_for, jsonify, g

app = Flask(__name__)
log = logging.getLogger('werkzeug')
log.setLevel(logging.ERROR)
app.config['SECRET_KEY'] = secrets.token_hex(32)

DB_PATH = 'chat.db'
MAX_MESSAGES_PER_ROOM = 100
MAX_MESSAGE_LENGTH = 1000


def get_db():
    if 'db' not in g:
        g.db = sqlite3.connect(DB_PATH)
        g.db.row_factory = sqlite3.Row  # доступ по имени колонки
    return g.db


@app.teardown_appcontext
def close_db(exception):
    db = g.pop('db', None)
    if db is not None:
        db.close()


def init_db():
    """Создаёт таблицы, если их нет."""
    conn = sqlite3.connect(DB_PATH)
    cur = conn.cursor()
    # Таблица комнат: код комнаты (уникальный), дата создания
    cur.execute("""
        CREATE TABLE IF NOT EXISTS rooms (
            code TEXT PRIMARY KEY,
            created_at TEXT NOT NULL
        )
    """)
    # Таблица сообщений: внешний ключ на комнату, автоинкремент ID
    cur.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            room_code TEXT NOT NULL,
            sender TEXT NOT NULL,
            message TEXT NOT NULL,
            timestamp TEXT NOT NULL,
            FOREIGN KEY (room_code) REFERENCES rooms(code) ON DELETE CASCADE
        )
    """)
    conn.commit()
    conn.close()


def generate_room_code(length=6):
    chars = string.ascii_uppercase + string.digits
    while True:
        code = ''.join(random.choice(chars) for _ in range(length))
        # Проверка на уникальность прямо в БД
        conn = sqlite3.connect(DB_PATH)
        cur = conn.cursor()
        cur.execute("SELECT 1 FROM rooms WHERE code = ?", (code,))
        exists = cur.fetchone()
        conn.close()
        if not exists:
            return code


@app.route('/V', methods=["GET", "POST"])
def home():
    error = None
    code_display = ''  # Переименовал в code_display, чтобы не путать с переменной кода комнаты

    # НЕ УДАЛЯЙ session.clear() здесь! Просто не используй его для сброса при GET.
    # Если нужно очищать сессию только при выходе, делай это в отдельной функции logout.
    
    if request.method == "POST":
        name = request.form.get('name', '').strip()
        create_pressed = request.form.get('create')
        join_pressed = request.form.get('join')
        room_code_input = request.form.get('code', '').strip().upper()

        # Проверка имени
        if not name:
            error = "Имя обязательно"
        else:
            # ЛОГИКА СОЗДАНИЯ КОМНАТЫ
            if create_pressed:
                # Игнорируем всё, что было в поле code. Генерируем новый код.
                new_room_code = generate_room_code()
                now = datetime.now().isoformat()
                
                conn = sqlite3.connect(DB_PATH)
                cur = conn.cursor()
                cur.execute("INSERT INTO rooms (code, created_at) VALUES (?, ?)", (new_room_code, now))
                conn.commit()
                conn.close()
                
                session['room'] = new_room_code
                session['name'] = name
                return redirect(url_for('room'))

            # ЛОГИКА ВХОДА В КОМНАТУ
            elif join_pressed:
                if not room_code_input:
                    error = "Введите код комнаты для входа"
                else:
                    conn = sqlite3.connect(DB_PATH)
                    cur = conn.cursor()
                    cur.execute("SELECT code FROM rooms WHERE code = ?", (room_code_input,))
                    row = cur.fetchone()
                    conn.close()
                    
                    if not row:
                        error = "Такой комнаты не существует"
                    else:
                        session['room'] = room_code_input
                        session['name'] = name
                        return redirect(url_for('room'))
            
            # Эта ветка теперь никогда не сработает, если есть кнопки create/join

    return render_template('home.html', error=error, code=code_display)


@app.route('/room')
def room():
    room_id = session.get('room')
    name = session.get('name')
    
    # Проверка сессии
    if not room_id or not name:
        return redirect(url_for('home'))

    # ИСПРАВЛЕНИЕ: Используем get_db(), а не создаем новое соединение!
    conn = get_db()
    cur = conn.cursor()
    
    try:
        cur.execute("""
            SELECT id, sender, message, timestamp
            FROM messages
            WHERE room_code = ?
            ORDER BY id ASC
            LIMIT ?
        """, (room_id, MAX_MESSAGES_PER_ROOM))
        
        rows = cur.fetchall()
        
        messages = [
            {
                "id": row["id"],
                "sender": row["sender"],
                "message": row["message"],
                "timestamp": row["timestamp"]
            }
            for row in rows
        ]
        return render_template('room.html', room=room_id, user=name, messages=messages)
    except Exception as e:
        # Для отладки можно вывести ошибку в консоль, но пользователю лучше показать страницу ошибки
        print(f"Error in room route: {e}")
        return redirect(url_for('home'))




@app.route('/send', methods=['POST'])
def send_message():
    data = request.get_json(silent=True) or {}
    room = data.get('room', '').strip()
    sender = data.get('name', '').strip()
    msg = data.get('message', '').strip()

    if not all([room, sender, msg]):
        return jsonify({"error": "Missing data"}), 400
    if len(msg) > MAX_MESSAGE_LENGTH:
        return jsonify({"error": "Message too long"}), 400

    now = datetime.now().strftime("%H:%M")
    conn = sqlite3.connect(DB_PATH)
    try:
        cur = conn.cursor()
        # Вставляем сообщение
        cur.execute(
            "INSERT INTO messages (room_code, sender, message, timestamp) VALUES (?, ?, ?, ?)",
            (room, sender, msg, now)
        )
        msg_id = cur.lastrowid

        # Удаляем старые сообщения, чтобы оставить не более MAX_MESSAGES_PER_ROOM
        cur.execute("""
            DELETE FROM messages
            WHERE room_code = ?
              AND id NOT IN (
                  SELECT id FROM messages
                  WHERE room_code = ?
                  ORDER BY id DESC
                  LIMIT ?
              )
        """, (room, room, MAX_MESSAGES_PER_ROOM))

        conn.commit()

        record = {
            "id": msg_id,
            "sender": sender,
            "message": msg,
            "timestamp": now
        }
        return jsonify(record)
    except Exception:
        conn.rollback()
        return jsonify({"error": "Database error"}), 500
    finally:
        conn.close()

@app.route('/about')
def about():
    return render_template('about.html')

@app.route('/messages/<room_code>')
def get_messages(room_code):
    last_id_str = request.args.get('last_id')
    last_id = None
    
    if last_id_str is not None:
        try:
            last_id = int(last_id_str)
        except ValueError:
            pass

    # ИСПРАВЛЕНИЕ: Используем get_db() вместо прямого connect
    conn = get_db()
    cur = conn.cursor()
    
    if last_id is not None:
        cur.execute(
            "SELECT id, sender, message, timestamp FROM messages WHERE room_code = ? AND id > ? ORDER BY id ASC",
            (room_code, last_id)
        )
    else:
        cur.execute(
            "SELECT id, sender, message, timestamp FROM messages WHERE room_code = ? ORDER BY id ASC LIMIT ?",
            (room_code, MAX_MESSAGES_PER_ROOM)
        )

    rows = cur.fetchall()

    messages = [
        {
            "id": row["id"],
            "sender": row["sender"],
            "message": row["message"],
            "timestamp": row["timestamp"]
        }
        for row in rows
    ]
    return jsonify(messages)



if __name__ == '__main__':
    init_db()  # инициализируем БД при старте
    webbrowser.open("http://192.168.0.101:8000/V")
    app.run(host='0.0.0.0', port=8000, debug=False)
