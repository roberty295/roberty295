

```python
from flask import Flask, render_template, request, redirect, url_for, session
import sqlite3
import hashlib

app = Flask(__wpion__)
app.secret_key = "segredo"

# Configuração do banco de dados
conn = sqlite3.connect('database.db')
cursor = conn.cursor()
cursor.execute('''CREATE TABLE IF NOT EXISTS tasks 
                  (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  task TEXT NOT NULL,
                  completed INTEGER NOT NULL DEFAULT 0,
                  user_id INTEGER,
                  FOREIGN KEY (user_id) REFERENCES users(id))''')
cursor.execute('''CREATE TABLE IF NOT EXISTS users 
                  (id INTEGER PRIMARY KEY AUTOINCREMENT,
                  username TEXT NOT NULL,
                  password TEXT NOT NULL)''')
conn.commit()

# Função para criptografar a senha do usuário
def encrypt_password(password):
    encrypted_password = hashlib.sha256(password.encode()).hexdigest()
    return encrypted_password

# Rota de registro de usuário
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        encrypted_password = encrypt_password(password)
        
        cursor.execute('INSERT INTO users (username, password) VALUES (?, ?)', (username, encrypted_password))
        conn.commit()
        
        return redirect(url_for('login'))
    return render_template('register.html')

# Rota de login
@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'POST':
        username = request.form['username']
        password = request.form['password']
        encrypted_password = encrypt_password(password)
        
        cursor.execute('SELECT * FROM users WHERE username=? AND password=?', (username, encrypted_password))
        user = cursor.fetchone()
        
        if user is not None:
            session['user_id'] = user[0]
            return redirect(url_for('tasks'))
    return render_template('login.html')

# Rota de logout
@app.route('/logout')
def logout():
    session.pop('user_id', None)
    return redirect(url_for('login'))

# Rota principal que exibe as tarefas do usuário logado
@app.route('/tasks')
def tasks():
    user_id = session.get('user_id')
    if user_id is None:
        return redirect(url_for('login'))
    
    cursor.execute('SELECT * FROM tasks WHERE user_id=?', (user_id,))
    tasks = cursor.fetchall()
    
    return render_template('tasks.html', tasks=tasks)

# Rota para adicionar uma nova tarefa
@app.route('/add_task', methods=['POST'])
def add_task():
    user_id = session.get('user_id')
    if user_id is None:
        return redirect(url_for('login'))
    
    task = request.form['task']
    
    cursor.execute('INSERT INTO tasks (task, user_id) VALUES (?, ?)', (task, user_id))
    conn.commit()
    
    return redirect(url_for('tasks'))

# Rota para marcar uma tarefa como concluída
@app.route('/complete_task/<int:task_id>')
def complete_task(task_id):
    user_id = session.get('user_id')
    if user_id is None:
        return redirect(url_for('login'))
    
    cursor.execute('UPDATE tasks SET completed=1 WHERE id=? AND user_id=?', (task_id, user_id))
    conn.commit()
    
    return redirect(url_for('tasks'))

# Rota para excluir uma tarefa
@app.route('/delete_task/<int:task_id>')
def delete_task(task_id):
    user_id = session.get('user_id')
    if user_id is None:
        return redirect(url_for('login'))
    
    cursor.execute('DELETE FROM tasks WHERE id=? AND user_id=?', (task_id, user_id))
    conn.commit()
    
    return redirect(url_for('tasks'))

if __name__ == '__main__':
    app.run(debug=True)
```

Explicação:
- O projeto utiliza a biblioteca Flask para criar um servidor web. As rotas `/register`, `/login`, `/logout`, `/tasks`, `/add_task`, `/complete_task` e `/delete_task` são definidas no arquivo `app.py`.
- O banco de dados SQLite é configurado no início do arquivo `app.py`. As tabelas `users` e `tasks` são criadas, caso não existam.
- Para cadastrar um usuário, a função `register()` é chamada quando a rota `/register` é acessada. O nome de usuário e a senha são inseridos na tabela `users`.
- A função `encrypt_password()` é utilizada para criptografar a senha do usuário antes de inseri-la na tabela `users`.
- Ao fazer login, a função `login()` é chamada quando a rota `/login` é acessada. O nome de usuário e a senha são verificados na tabela `users`. Se as credenciais estiverem corretas, o ID do usuário é armazenado na sessão e o redirecionamento é feito para a rota `/tasks`.
- A função `tasks()` é chamada quando a rota `/tasks` é acessada. Primeiro, verifica se o ID do usuário está armazenado na sessão. Se estiver, as tarefas relacionadas ao usuário são buscadas na tabela `tasks` e renderizadas na página `tasks.html` junto com o template HTML.
- Para adicionar uma nova tarefa, a função `add_task()` é chamada quando a rota `/add_task` é acessada. O texto da tarefa é enviado via post e é inserido na tabela `tasks`, juntamente com o ID do usuário logado.
- Para marcar uma tarefa como concluída, a função `complete_task()` é chamada quando a rota `/complete_task/<int:task_id>` é acessada. O ID da tarefa é passado como parâmetro e a coluna `completed` da tabela `tasks` é atualizada para 1.
- Para excluir uma tarefa, a função `delete_task()` é chamada quando a rota `/delete_task/<int:task_id>` é acessada. O ID da tarefa é passado como parâmetro e a tarefa é removida da tabela `tasks`.

