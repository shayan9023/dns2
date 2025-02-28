#!/bin/bash
# Unbound DNS Server Installer & Web Panel Setup

# Update & Install Required Packages
echo "Updating system and installing dependencies..."
sudo apt update && sudo apt install -y unbound python3 python3-pip sqlite3

# Download Root Hints
echo "Downloading Root Hints..."
sudo curl -o /etc/unbound/root.hints https://www.internic.net/domain/root.hints

# Configure Unbound
echo "Configuring Unbound..."
cat <<EOF | sudo tee /etc/unbound/unbound.conf
server:
    interface: 0.0.0.0
    access-control: 0.0.0.0/0 allow
    verbosity: 1
    do-ip6: no
    root-hints: "/etc/unbound/root.hints"
    hide-identity: yes
    hide-version: yes
EOF

# Restart and Enable Unbound
echo "Starting Unbound service..."
sudo systemctl restart unbound
sudo systemctl enable unbound

# Setup Web Panel
mkdir -p /opt/dns_manager
cd /opt/dns_manager

# Create Database
echo "Setting up database..."
cat <<EOF > setup_db.py
import sqlite3
import secrets
conn = sqlite3.connect('dns_users.db')
c = conn.cursor()
c.execute('''CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username TEXT, token TEXT UNIQUE, data_limit INTEGER, time_limit INTEGER, created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP)''')
c.execute('''CREATE TABLE IF NOT EXISTS blocked_users (token TEXT PRIMARY KEY)''')
conn.commit()
conn.close()
EOF
python3 setup_db.py

# Install Flask & Create Web Panel
echo "Installing Flask and setting up web panel..."
pip3 install flask flask_sqlalchemy

cat <<EOF > web_panel.py
from flask import Flask, request, jsonify, render_template
import sqlite3
import secrets
import time

app = Flask(__name__)

def db_connection():
    conn = sqlite3.connect('dns_users.db')
    return conn

@app.route('/')
def home():
    return render_template('index.html')

@app.route('/add_user', methods=['POST'])
def add_user():
    data = request.json
    token = secrets.token_hex(16)  # Generate a unique token
    data_limit = data.get('data_limit', 0)
    time_limit = data.get('time_limit', 0)
    conn = db_connection()
    c = conn.cursor()
    c.execute("INSERT INTO users (username, token, data_limit, time_limit) VALUES (?, ?, ?, ?)",
              (data['username'], token, data_limit, time_limit))
    conn.commit()
    conn.close()
    return jsonify({"message": "User added successfully!", "token": token})

@app.route('/list_users', methods=['GET'])
def list_users():
    conn = db_connection()
    c = conn.cursor()
    c.execute("SELECT username, token, data_limit, time_limit FROM users")
    users = c.fetchall()
    conn.close()
    return jsonify(users)

@app.route('/delete_user', methods=['POST'])
def delete_user():
    data = request.json
    username = data['username']
    conn = db_connection()
    c = conn.cursor()
    
    # Get user's token before deleting
    c.execute("SELECT token FROM users WHERE username = ?", (username,))
    user_token = c.fetchone()
    
    if user_token:
        user_token = user_token[0]
        
        # Add user to blocked list
        c.execute("INSERT INTO blocked_users (token) VALUES (?)", (user_token,))
        conn.commit()
        
        # Remove user from main users table
        c.execute("DELETE FROM users WHERE username = ?", (username,))
        conn.commit()
        conn.close()
        
        return jsonify({"message": f"User {username} deleted and blocked from DNS."})
    else:
        conn.close()
        return jsonify({"message": "User not found."}), 404

@app.route('/validate_token', methods=['POST'])
def validate_token():
    data = request.json
    token = data['token']
    conn = db_connection()
    c = conn.cursor()
    
    # Check if token exists and is not blocked
    c.execute("SELECT * FROM users WHERE token = ?", (token,))
    user = c.fetchone()
    c.execute("SELECT * FROM blocked_users WHERE token = ?", (token,))
    blocked = c.fetchone()
    conn.close()
    
    if user and not blocked:
        return jsonify({"message": "Valid token"}), 200
    else:
        return jsonify({"message": "Invalid or blocked token"}), 403

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create HTML UI
echo "Creating web panel UI..."
mkdir -p templates
cat <<EOF > templates/index.html
<!DOCTYPE html>
<html>
<head>
    <title>DNS Management Panel</title>
</head>
<body>
    <h2>DNS Management Panel</h2>
    <form action="/add_user" method="post">
        <label>Username:</label>
        <input type="text" name="username" required>
        <label>Data Limit (MB, 0 for unlimited):</label>
        <input type="number" name="data_limit" min="0">
        <label>Time Limit (seconds, 0 for unlimited):</label>
        <input type="number" name="time_limit" min="0">
        <button type="submit">Create User</button>
    </form>
    <h3>Users:</h3>
    <div id="user-list"></div>
    <script>
        fetch('/list_users')
            .then(response => response.json())
            .then(data => {
                let userList = document.getElementById('user-list');
                data.forEach(user => {
                    userList.innerHTML += `<p>${user[0]} - Token: ${user[1]}</p>`;
                });
            });
    </script>
</body>
</html>
EOF

# Start Web Panel
echo "Starting Web Panel..."
nohup python3 web_panel.py &

echo "Installation complete! Web panel is running on port 5000."
