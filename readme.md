To integrate SQLite into your ChainLit chatbot and store all messages (AI, human, tool, and system), you'll need to:

1. Connect to an SQLite database


2. Create a table to store messages


3. Capture and insert messages during the conversation


4. Use ChainLit’s session ID as a unique key




---

✅ Step-by-Step Implementation

1. Database Setup

# db.py
import sqlite3

DB_PATH = "chat_history.db"

def get_connection():
    return sqlite3.connect(DB_PATH)

def create_table():
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            session_id TEXT NOT NULL,
            role TEXT CHECK(role IN ('human', 'ai', 'tool', 'system')),
            content TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()


---

2. ChainLit Hooks to Store Messages

ChainLit provides message lifecycle hooks. We’ll use them to intercept messages and store them.

# main.py
import chainlit as cl
from db import create_table, get_connection

create_table()

def store_message(session_id: str, role: str, content: str):
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO messages (session_id, role, content) VALUES (?, ?, ?)",
        (session_id, role, content)
    )
    conn.commit()
    conn.close()

@cl.on_message
async def handle_user_message(message: cl.Message):
    session_id = cl.user_session.id
    store_message(session_id, "human", message.content)

    # Simulate processing
    response = f"Echo: {message.content}"
    await cl.Message(content=response).send()

    store_message(session_id, "ai", response)

@cl.on_chat_start
def on_chat_start():
    session_id = cl.user_session.id
    store_message(session_id, "system", "Chat session started.")

@cl.on_tool_message
async def on_tool_message(message: cl.Message):
    session_id = cl.user_session.id
    store_message(session_id, "tool", message.content)


---

✅ Resulting SQLite Table Schema

id	session_id	role	content	timestamp

1	abc123	human	Hello	2025-07-08 10:00:00
2	abc123	ai	Echo: Hello	2025-07-08 10:00:01
3	abc123	tool	Tool result here	2025-07-08 10:00:02



---

🛠 Notes

Use cl.user_session.id as your session key.

You can enhance the schema with tool_name, message_type, or meta_json if needed.

You can retrieve the full history per session using:


SELECT * FROM messages WHERE session_id = ? ORDER BY timestamp


---

Would you like me to also provide a function to retrieve full chat history for a given session from SQLite?
