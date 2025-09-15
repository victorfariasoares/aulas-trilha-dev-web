# Aula 4

Até a aula passada, estávamos guardando os post-its apenas em memória, dentro de uma lista Python (`notes`) algo extremamente simples.
Agora é hora de usar um **banco de dados de verdade** para que os dados fiquem **persistentes**, ou seja, não desapareçam quando o servidor reiniciar.

## Objetivos

Os nossos objetivos nesta aula estão centralizados em implementar toda a estrutura necessária para usar um banco de dados PostgreSQL com Flask, e também adicionar mais uma funcionalidade, a de **Editar** um post-it já existente.

1. Instalar e configurar **PostgreSQL** e **pgAdmin**.
2. Conectar o back-end Flask ao banco SQLAlchemy (ORM — mapear tabelas em classes Python).
3. Criar tabela e migrar os endpoints (`GET`, `POST`, `DELETE`).
4. Implementar o **Editar (Update)** com rota `PUT /api/notes/<id>`.
5. Criar a página de edição no front-end (`EditNote.jsx`).

---


# 1) Instalando PostgreSQL + pgAdmin

Até agora trabalhamos com dados em **memória** (lista Python), mas isso não é suficiente para uma aplicação real.
Por isso, vamos usar o **PostgreSQL**, um dos bancos de dados relacionais mais usados no mundo.

Mas além do **servidor PostgreSQL** (onde os dados ficam armazenados de fato), também vamos instalar o **pgAdmin**.

---

## O que é o pgAdmin?

O **pgAdmin** é uma ferramenta gráfica oficial para gerenciar o PostgreSQL.
Ele funciona como um **painel de controle visual** para o banco de dados:

* Permite **criar bancos e tabelas** sem precisar decorar comandos SQL.
* Facilita a **execução de queries** (SELECT, INSERT, UPDATE, DELETE) com um editor embutido.
* Exibe os dados em **tabelas visuais**, o que ajuda a entender melhor o que está acontecendo.
* Permite **monitorar conexões e desempenho**.

O pgAdmin não é a única forma de gerenciar PostgreSQL. Alguns exemplos populares:

* **psql** → Cliente oficial em linha de comando. Vem instalado junto com o PostgreSQL.
* **DBeaver** → Ferramenta gráfica universal (funciona com PostgreSQL, MySQL, Oracle, SQLite e muitos outros).
* **TablePlus** → Interface paga (mas tem versão gratuita limitada), bem moderna e leve.
* **DataGrip** → IDE profissional da JetBrains, suporta vários bancos (PostgreSQL, MySQL, MongoDB etc.).
* **HeidiSQL / Beekeeper Studio** → Outras opções conhecidas para gerenciar bancos SQL.

> Aqui vamos usar o **pgAdmin** porque é gratuito, oficial e suficiente para o nosso projeto.
> Mas em projetos profissionais você pode acabar usando **DBeaver** ou **DataGrip**, que são mais completos.



### Windows

Baixe o instalador em https://www.pgadmin.org/ 

E [aqui](https://www.youtube.com/watch?v=gEgRSS3Z8JQ) tem um vídeo da própria matéria de TecWeb mostrando o passo a passo.

Crie com a senha `admin` (anote essa senha, você vai precisar dela para entrar no pgAdmin).

### macOS

```bash
brew install postgresql
brew services start postgresql
```


### Linux (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

---

# 2) Criando banco

## 2.1 Abra o **pgAdmin**

## 2.2 Clique com botão direito em Databases → Create → Database.

**Nome**: trilha_db

**Dono**: postgres.

**Salvar**.

> **Importante:** nesta abordagem não precisamos criar tabelas manualmente.
> As tabelas serão geradas automaticamente a partir dos **modelos Python** usando migrations.
---

# 3) Configurando o back-end com SQLAlchemy

### 3.1 Instalar dependências

No **venv-back**, rode:

```bash
pip install Flask-SQLAlchemy Flask-Migrate psycopg2-binary python-dotenv
```

E atualize o `requirements.txt`:
```bash
pip freeze > requirements.txt
```

---

### 3.2 Estrutura de pastas

Até a Aula 3, todo o código do back-end estava dentro de um único arquivo (`app.py`).
Isso funciona em projetos pequenos, mas logo fica difícil de manter: o arquivo cresce demais, mistura responsabilidades (rotas, conexão com banco, regras de negócio), e qualquer alteração vira uma bagunça.

Por isso, agora separamos em arquivos e pastas bem definidos:

```
backend/
├─ app.py             # Ponto de entrada da aplicação
├─ app/
│  ├─ __init__.py     # Criação da app Flask + integração com extensões
│  ├─ config.py       # Configurações (ex.: banco de dados, variáveis de ambiente)
│  ├─ models.py       # Modelos (classes que viram tabelas no banco)
│  └─ routes.py       # Rotas (endpoints da API)
└─ migrations/        # Histórico de alterações no banco (será criado pelo Flask-Migrate)

```

---

### 3.3 Configuração (`app/config.py`)

Guarda as informações sensíveis e de ambiente (como URL do banco e chave secreta):

```python
import os
from dotenv import load_dotenv

load_dotenv()

class Config:
    SQLALCHEMY_DATABASE_URI = os.getenv("DATABASE_URL")
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

Crie também um arquivo **.env** na pasta `backend/`:

```env
DATABASE_URL=postgresql://postgres:admin@localhost:5432/trilha_db
SECRET_KEY=umasecretkey
```

O formato da conexão é sempre:
```bash
postgresql://USUARIO:SENHA@HOST:PORTA/NOME_DO_BANCO
```
`SECRET_KEY`

Essa chave é usada pelo Flask para **assinar tokens e cookies de sessão**.

* Você mesmo **define qualquer valor aleatório**.
* No seu caso, pode ser `"umasecretkey"`, `"segredo123"`, `"minha_chave_super_segura"`, ou até gerar algo maior.
* Não precisa ter definido antes, é você que decide agora.
* Em projetos sérios, o ideal é gerar algo bem forte (ex.: `openssl rand -hex 32`) e **não compartilhar no GitHub** — sempre deixar no `.env`.

Neste projeto, o `SECRET_KEY` ainda não está sendo usado diretamente, mas já deixamos ele preparado para quando precisar (ex.: autenticação com JWT ou sessões Flask).

---

### 3.4 Inicialização (`app/__init__.py`)

Aqui centralizamos as extensões do Flask:
- `db = SQLAlchemy()` → conecta o Flask ao banco de dados.

- `migrate = Migrate()` → gerencia migrations (criação e alteração de tabelas).

- `create_app()` → cria a instância Flask e registra o blueprint com as rotas.

Esse arquivo é o coração do back-end: garante que tudo esteja pronto (banco, CORS, rotas).

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy
from flask_migrate import Migrate
from flask_cors import CORS
from .config import Config

db = SQLAlchemy()
migrate = Migrate()

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    CORS(app, origins=["http://localhost:5173"])

    db.init_app(app)
    migrate.init_app(app, db)

    from .routes import main
    app.register_blueprint(main)

    return app
```

---

### 3.5 Model (`app/models.py`)

Cada classe Python aqui representa uma tabela do banco:

```python
from . import db

class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(150), nullable=False)
    content = db.Column(db.Text, nullable=False)
    tag = db.Column(db.String(50))
```
`Note` → vira a tabela `note`.

`id` → coluna inteira, chave primária.

`title`, `content` → colunas obrigatórias (`nullable=False`).

`tag` → coluna opcional.

Graças ao SQLAlchemy, não precisamos escrever SQL manual.

---

### 3.6 Rotas (`app/routes.py`)

Aqui ficam os endpoints (as “portas de entrada” para o front se comunicar com o back).
Exemplos:
- `POST /api/notes` → cria uma nota.

- `GET /api/notes/<id>` → busca uma nota específica.

- `PUT /api/notes/<id>` → edita uma nota.

- `DELETE /api/notes/<id>` → apaga uma nota.

```python
from flask import Blueprint, request, jsonify
from .models import Note
from . import db

main = Blueprint("main", __name__)

# GET all
@main.route("/api/notes", methods=["GET"])
def get_notes():
    notes = Note.query.order_by(Note.id).all()
    return jsonify([{
        "id": n.id, "title": n.title, "content": n.content, "tag": n.tag
    } for n in notes])

# POST
@main.route("/api/notes", methods=["POST"])
def add_note():
    data = request.get_json()

    note = Note(
        title=data["title"].strip(),
        content=data["content"].strip(),
        tag=data.get("tag", "").strip()
    )
    db.session.add(note)
    db.session.commit()
    return jsonify({
        "id": note.id, "title": note.title, "content": note.content, "tag": note.tag
    }), 201

# GET one
@main.route("/api/notes/<int:id>", methods=["GET"])
def get_note(id):
    note = Note.query.get_or_404(id)
    return jsonify({"id": note.id, "title": note.title, "content": note.content, "tag": note.tag})

# PUT
@main.route("/api/notes/<int:id>", methods=["PUT"])
def update_note(id):
    note = Note.query.get_or_404(id)
    data = request.get_json()

    note.title = data["title"].strip()
    note.content = data["content"].strip()
    note.tag = data.get("tag", "").strip()
    db.session.commit()
    return jsonify({"id": note.id, "title": note.title, "content": note.content, "tag": note.tag})

# DELETE
@main.route("/api/notes/<int:id>", methods=["DELETE"])
def delete_note(id):
    note = Note.query.get_or_404(id)
    db.session.delete(note)
    db.session.commit()
    return "", 204
```

---

### 3.7 Arquivo principal (`app.py`)

### Atualizando o `app.py`

Até a Aula 3, o nosso `app.py` tinha **tudo dentro dele**:

* Era o banco de dados (lista `notes`);
* As rotas da API (`/api/notes`, `/api/tags` etc.);
* A função principal para rodar o servidor Flask.

Esse modelo **funciona em projetos pequenos**, mas rapidamente vira um problema:
o arquivo fica gigante, difícil de organizar e de manter.


---

### Como fica o `app.py` agora?

Apagamos todo o conteúdo antigo e deixamos apenas:

```python
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=5000)
```

---

### O que está acontecendo aqui?

* `from app import create_app` → Importa a função `create_app` que criamos em `app/__init__.py`.
* `app = create_app()` → Cria a instância do Flask já configurada com:

  * Conexão ao banco de dados (SQLAlchemy);
  * Sistema de migrations (Flask-Migrate);
  * CORS habilitado;
  * Blueprints com todas as rotas da API.
* O bloco `if __name__ == "__main__":` → Garante que o servidor Flask só seja iniciado quando rodamos `python app.py`, e não quando importamos esse arquivo em outro lugar.
* `app.run(...)` → De fato, sobe o servidor na porta `5000`.

### Por que mudar?

Agora que estamos usando **SQLAlchemy + Flask-Migrate**, é a hora de organizar o projeto de forma mais profissional:

* A **conexão com o banco** fica no `app/__init__.py`.
* As **rotas** ficam separadas em `app/routes.py`.
* Os **modelos do banco (tabelas)** ficam em `app/models.py`.
* O `app.py` passa a ser **somente o ponto de entrada** da aplicação.

Assim, o código fica mais limpo, fácil de entender e **pronto para crescer** (com várias tabelas e rotas diferentes).


---

# 4) Migrations (criando tabela notes)

O SQLAlchemy permite mapear tabelas como classes Python (ORM).
Com o **Flask-Migrate**, conseguimos criar a estrutura do banco diretamente a partir dos modelos.

No terminal, dentro da pasta `backend/`:

```bash
flask db init          # cria a pasta migrations (somente 1 vez no projeto)
flask db migrate -m "create notes"
flask db upgrade       # aplica as alterações → cria a tabela notes no banco
```

Agora, sempre que fizer alterações no `models.py`, o fluxo é:
```bash
flask db migrate -m "descricao da mudança"
flask db upgrade
```

1. **`flask db init`**

   * Criou a pasta `migrations/` e os arquivos de configuração do Alembic (que é quem o Flask-Migrate usa por baixo dos panos).
   * Isso só precisa ser feito **uma vez por projeto**.

2. **`flask db migrate -m "create notes"`**

   * O Flask olhou para o seu `models.py`, percebeu que existe um modelo `Note`, e gerou um script de migration (`ab1b3f962119_create_notes.py`) dentro de `migrations/versions/`.
   * Esse script é tipo um "histórico" dizendo: *"criar a tabela note com as colunas id, title, content, tag"*.

3. **`flask db upgrade`**

   * Aplicou esse script no banco de dados `trilha_db`.
   * Agora a tabela `note` foi realmente criada no PostgreSQL.



* Abra o **pgAdmin**, atualize as tabelas dentro de `trilha_db`, e você deve ver a tabela `note`.


* No **pgAdmin**, em `Tables`, aparecem:

  * `note` → que é justamente a tabela criada a partir do seu modelo `Note` em `models.py`.
  * `alembic_version` → essa tabela é criada automaticamente pelo Flask-Migrate para controlar quais migrations já foram aplicadas.


* Clique com o botão direito → **View/Edit Data → All Rows** e deve aparecer uma tabela vazia, mas já com as colunas (`id`, `title`, `content`, `tag`).


  * `id` (PK, auto-incremento)
  * `title` (`varchar(150)`, obrigatório)
  * `content` (`text`, obrigatório)
  * `tag` (`varchar(50)`, opcional)

Ou seja, está exatamente de acordo com o modelo que você definiu no `models.py`:

```python
class Note(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(150), nullable=False)
    content = db.Column(db.Text, nullable=False)
    tag = db.Column(db.String(50))
```


Agora temos uma tabela `notes` pronta para receber nossos dados.

Os tipos de dados que usamos em nossas tabelas são muito importantes, pois diferentes tipos de dados tem diferentes tratamentos e consomem diferentes quantidades de espaço em disco. Abaixo estão os principais tipos de dados de string em SQL.

![alt text](image.png)

## O que mudou da Aula 3 para a Aula 4?

Tente adicionar e deletar um post-it pela interface web e veja que ele aparece no banco de dados (pgAdmin).

Na Aula 3, nosso back-end usava apenas uma **lista em memória (`notes`)** para armazenar os post-its.
Agora, na Aula 4, fizemos a **migração para o PostgreSQL** usando **SQLAlchemy**.


---

# 5) Vamos criar agora a funcionalidade de Edição

## Página de edição

Crie **`src/pages/EditNote.jsx`**:

```jsx
import { useEffect, useState } from "react";
import { useParams, useNavigate } from "react-router-dom";

export default function EditNote() {
  const { id } = useParams();
  const navigate = useNavigate();
  const [title, setTitle] = useState("");
  const [content, setContent] = useState("");
  const [tag, setTag] = useState("");

  // Buscar nota existente pelo id
  useEffect(() => {
    fetch(`http://localhost:5000/api/notes/${id}`)
      .then((r) => r.json())
      .then((note) => {
        if (note.error) {
          alert("Nota não encontrada!");
          navigate("/");
        } else {
          setTitle(note.title);
          setContent(note.content);
          setTag(note.tag || "");
        }
      });
  }, [id, navigate]);

  function handleUpdate(e) {
    e.preventDefault();
    fetch(`http://localhost:5000/api/notes/${id}`, {
      method: "PUT",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title, content, tag }),
    })
      .then((r) => r.json())
      .then(() => navigate("/"));
  }

  return (
    <main className="container">
      <form className="form-card" onSubmit={handleUpdate}>
        <input
          className="form-card-title"
          value={title}
          onChange={(e) => setTitle(e.target.value)}
        />
        <textarea
          className="autoresize"
          value={content}
          onChange={(e) => setContent(e.target.value)}
        />
        <input
          className="form-card-tag"
          value={tag}
          onChange={(e) => setTag(e.target.value)}
        />
        <button className="btn-criar">Salvar</button>
      </form>
    </main>
  );
}
```

---

## Ajustando rotas

No **`routes.jsx`**, adicione a rota de edição:

```jsx
import EditNote from "../pages/EditNote";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,
    children: [
      { path: "/", element: <Home /> },
      { path: "/edit/:id", element: <EditNote /> },
    ],
  },
]);
```

---

## Botão de edição

No **`Home.jsx`**, dentro do `map`, adicione o botão de edição junto da lixeira:

```jsx
  <a href={`/edit/${note.id}`} className="edit">✏️</a>
```

---
Agora cada post-it terá **botão de editar** e **botão de deletar**. Teste para ver se deu certo.

O nosso fluxo ficou assim:

* `GET /api/notes` → lista todas as notas.
* `GET /api/notes/<id>` → busca só uma nota.
* `PUT /api/notes/<id>` → edita uma nota.
* O front consome diretamente o endpoint correto sem baixar dados desnecessários.

---

# 6) Correção de notas 

Por enquanto, ao tentar criar ou editar uma nota, não há validação se o título ou conteúdo estão vazios.
Até colocamos no SQL que esses campos são `NOT NULL`, mas se tentarmos enviar uma nota sem conteúdo ou título, ela vai ser adicionada normalmente pois fica como string vazia `""` o que é diferente de `NULL`.

## Editando o `routes.py`:

Adicione este campo em duas funções logo depois de `data = request.get_json()` no `/api/notes POST` e `/api/notes/<int:id> PUT`:

```python
if not data.get("title", "").strip() or not data.get("content", "").strip():
        return jsonify({"error": "Título e conteúdo são obrigatórios"}), 400
```

## Atualizando o front `Home.jsx`:

```jsx
	function handleSubmit(e) {
		e.preventDefault();
		
		if (!title.trim() || !content.trim()) {
			alert("Título e conteúdo são obrigatórios!");
			return;
		}
		fetch("http://localhost:5000/api/notes", {
			method: "POST",
			headers: { "Content-Type": "application/json" },
			body: JSON.stringify({ title, content, tag }),
		})
			.then(async (r) => {
				if (r.status === 201) {
					const newNote = await r.json();
					setNotes([...notes, newNote]); // só adiciona se o back realmente criou
					setTitle("");
					setContent("");
					setTag("");
				} else if (r.status === 400) {
					const error = await r.json();
					alert(error.error);
				}
			});
	}
```

## Atualizando o `EditNote.jsx`:

Mesma coisa que fizemos em `Home.jsx`, mas na função `handleUpdate`:

```jsx
function handleUpdate(e) {
  e.preventDefault();

  // Validação no front: impedir edição vazia
  if (!title.trim() || !content.trim()) {
    alert("Título e conteúdo são obrigatórios!");
    return;
  }

  fetch(`http://localhost:5000/api/notes/${id}`, {
    method: "PUT",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ title, content, tag }),
  })
    .then(async (r) => {
      if (r.status === 200) {
        await r.json();
        navigate("/");
      } else if (r.status === 400) {
        const error = await r.json();
        alert(error.error);
      }
    });
}

```

# 7) Validação

Já implementamos no back e no front a checagem para evitar notas sem título ou conteúdo. Confira se ao tentar criar ou editar uma nota vazia, aparece o alerta.

---


# 8) Trabalhando com Tags

Até agora, nossas notas já tinham um campo `tag`, mas ele ainda não estava sendo usado de forma completa.
Agora vamos implementar mas sem CSS:

1. **Endpoint para listar todas as tags existentes.**
2. **Página `/tags` para mostrar a lista de tags.**
3. **Página `/tags/:tag` para mostrar todas as notas de uma tag específica.**
4. **Links nas notas para navegar por tag.**

---

## 5.1 Back-end (Flask)

### a) Listar todas as tags

No `routes.py`, adicione:

```python
# Listar todas as tags únicas
@main.route("/api/tags", methods=["GET"])
def list_tags():
    tags = db.session.query(Note.tag).distinct().filter(Note.tag != "").all()
    return jsonify([t[0] for t in tags])
```

---

### b) Listar notas de uma tag específica

Ainda no `routes.py`, adicione:

```python
# Listar todas as notas de uma tag específica
@main.route("/api/tags/<tag_name>", methods=["GET"])
def notes_by_tag(tag_name):
    notes = Note.query.filter_by(tag=tag_name).all()
    return jsonify([
        {"id": n.id, "title": n.title, "content": n.content, "tag": n.tag}
        for n in notes
    ])
```

---

## 5.2 Front-end — Página de Tags

Crie **`src/pages/Tags.jsx`**:

```jsx
import { useEffect, useState } from "react";
import { Link } from "react-router-dom";

export default function Tags() {
  const [tags, setTags] = useState([]);

  useEffect(() => {
    fetch("http://localhost:5000/api/tags")
      .then((r) => r.json())
      .then(setTags);
  }, []);

  return (
    <main className="container">
      <h2>Todas as tags</h2>
      <ul>
        {tags.map((t, i) => (
          <li key={i}>
            <Link to={`/tags/${t}`} className="card-tag-link">#{t}</Link>
          </li>
        ))}
      </ul>
    </main>
  );
}
```

---

## 5.3 Front-end — Página de Notas por Tag

Crie **`src/pages/NotesByTag.jsx`**:

```jsx
import { useEffect, useState } from "react";
import { useParams, Link } from "react-router-dom";

export default function NotesByTag() {
  const { tag } = useParams();
  const [notes, setNotes] = useState([]);

  useEffect(() => {
    fetch(`http://localhost:5000/api/tags/${tag}`)
      .then((r) => r.json())
      .then(setNotes);
  }, [tag]);

  return (
    <main className="container">
      <h2>Notas com a tag: #{tag}</h2>
      <ul className="card-container">
        {notes.map((note) => (
          <div className="card" key={note.id}>
            <h3 className="card-title">{note.title}</h3>
            <div className="card-content">
              <p>{note.content}</p>
              {note.tag && (
                <Link to={`/tags/${note.tag}`} className="card-tag">
                  #{note.tag}
                </Link>
              )}
            </div>
          </div>
        ))}
      </ul>
    </main>
  );
}
```

---

## 5.4 Ajustando rotas

No **`routes.jsx`**, adicione as duas páginas novas:

```jsx
import Tags from "../pages/Tags";
import NotesByTag from "../pages/NotesByTag";

export const router = createBrowserRouter([
  {
    path: "/",
    element: <Layout />,
    children: [
      { path: "/", element: <Home /> },
      { path: "/edit/:id", element: <EditNote /> },
      { path: "/tags", element: <Tags /> },
      { path: "/tags/:tag", element: <NotesByTag /> },
    ],
  },
]);
```

---

## 5.5 Linkando tags nas notas

No **`Home.jsx`**, ajuste a parte que renderiza a tag para ficar clicável:

```jsx
{note.tag && (
  <a href={`/tags/${note.tag}`} className="card-tag-link">#{note.tag}</a>
)}
```

---

## 5.6 Resultado:

1. Criar notas com tags.
2. Ir para `/tags` → lista de todas as tags únicas.
3. Clicar em uma tag → `/tags/:tag` → mostra só as notas daquela tag.
---

# 9) Conclusão

Agora temos:

* Back-end organizado com **Flask + SQLAlchemy + Migrations**;
* Modelo `Note` que representa a tabela `notes`;
* CRUD completo (Create, Read, Update, Delete);
* **Validação** para evitar notas vazias;
* **Tags** para organizar e filtrar notas.




