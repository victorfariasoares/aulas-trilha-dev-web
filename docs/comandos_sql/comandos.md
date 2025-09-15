

# 📘 Guia de SQL (PostgreSQL)



## 1. Bancos de Dados

```sql
-- Criar um novo banco de dados
CREATE DATABASE trilha_db;

-- Apagar banco de dados (⚠️ remove tudo)
DROP DATABASE trilha_db;

-- Conectar em um banco (pelo psql)
\c trilha_db;
```

---

## 2. Tabelas

### Criar tabela

```sql
CREATE TABLE notes (
    id SERIAL PRIMARY KEY,           -- chave primária, auto-incremental
    title VARCHAR(200) NOT NULL,     -- string de até 200 caracteres
    content TEXT NOT NULL,           -- texto sem limite definido
    tag VARCHAR(50),                 -- string curta
    created_at TIMESTAMP DEFAULT NOW() -- data/hora automática
);
```

Você pode verificar todos os tipos de dados [aqui](https://www.postgresql.org/docs/current/datatype.html) ou [W3 Schools](https://www.w3schools.com/sql/sql_datatypes.asp).

### Apagar tabela

```sql
DROP TABLE notes;           -- apaga a tabela
DROP TABLE IF EXISTS notes; -- apaga só se existir
```

### Alterar tabela existente

```sql
-- Renomear tabela
ALTER TABLE notes RENAME TO postits;

-- Adicionar coluna
ALTER TABLE notes ADD COLUMN author VARCHAR(100);

-- Alterar tipo de coluna
ALTER TABLE notes ALTER COLUMN title TYPE VARCHAR(300);

-- Apagar coluna
ALTER TABLE notes DROP COLUMN author;
```

---

## 3. Inserindo Dados

```sql
-- Inserir valores em todas as colunas (menos o id, que é automático)
INSERT INTO notes (title, content, tag)
VALUES ('Primeira nota', 'Esse é o conteúdo da nota', 'inicio');

-- Inserir múltiplas linhas de uma vez
INSERT INTO notes (title, content, tag)
VALUES
  ('Estudar SQL', 'Praticar comandos básicos', 'estudo'),
  ('Reunião', 'Preparar slides até sexta', 'trabalho'),
  ('Compras', 'Leite, pão, café', 'pessoal');
```

---

## 4. Consultando Dados

```sql
-- Selecionar tudo
SELECT * FROM notes;

-- Selecionar colunas específicas
SELECT id, title FROM notes;

-- Renomear colunas no resultado
SELECT id AS codigo, title AS titulo FROM notes;

-- Selecionar com condição
SELECT * FROM notes WHERE tag = 'estudo';

-- Condições múltiplas
SELECT * FROM notes WHERE tag = 'estudo' AND title LIKE '%SQL%';

-- Operadores comuns:
-- =   igual
-- <>  diferente
-- >   maior
-- <   menor
-- >=  maior ou igual
-- <=  menor ou igual
-- LIKE  comparação com padrões
-- ILIKE comparação sem diferenciar maiúsc/minúsc

-- Exemplo com LIKE (buscar por "SQL" em qualquer parte do título)
SELECT * FROM notes WHERE title LIKE '%SQL%';

-- Ordenar resultados
SELECT * FROM notes ORDER BY created_at DESC;

-- Limitar resultados
SELECT * FROM notes LIMIT 5;

-- Pular resultados e trazer só os próximos (paginação)
SELECT * FROM notes OFFSET 5 LIMIT 5;
```

---

## 5. Atualizando Dados

```sql
-- Atualizar uma linha específica
UPDATE notes
SET content = 'Novo conteúdo da nota'
WHERE id = 1;

-- Atualizar várias linhas de uma vez
UPDATE notes
SET tag = 'pessoal'
WHERE tag IS NULL;
```

---

## 6. Deletando Dados

```sql
-- Apagar uma nota específica
DELETE FROM notes WHERE id = 2;

-- Apagar todas as notas com tag "pessoal"
DELETE FROM notes WHERE tag = 'pessoal';

-- Apagar todos os registros da tabela
DELETE FROM notes;
```

---

## 7. Funções Úteis

```sql
-- Contar registros
SELECT COUNT(*) FROM notes;

-- Maior e menor id
SELECT MAX(id), MIN(id) FROM notes;

-- Quantidade de notas por tag
SELECT tag, COUNT(*) FROM notes GROUP BY tag;

-- Nota mais recente
SELECT * FROM notes ORDER BY created_at DESC LIMIT 1;
```

---

## 8. Chaves Estrangeiras (Relacionamentos)

Exemplo: um post-it pertence a um usuário.

```sql
-- Criar tabela de usuários
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL
);

-- Alterar tabela notes para relacionar com users
ALTER TABLE notes
ADD COLUMN user_id INT REFERENCES users(id);
```

Agora cada nota pode ter um `user_id` que aponta para a tabela `users`.

---

## 9. Joins (Relacionando Tabelas)

```sql
-- Juntar notas com seus usuários
SELECT n.id, n.title, u.name
FROM notes n
JOIN users u ON n.user_id = u.id;

-- Trazer todas as notas, mesmo sem usuário
SELECT n.id, n.title, u.name
FROM notes n
LEFT JOIN users u ON n.user_id = u.id;
```


---

## 10. `BETWEEN`

A cláusula **`BETWEEN`** é usada em SQL para selecionar valores **dentro de um intervalo**, inclusive os limites.

### Sintaxe:

```sql
valor BETWEEN minimo AND maximo
```

É equivalente a:

```sql
valor >= minimo AND valor <= maximo
```

---

### Exemplos práticos

#### 1. Notas com `id` entre 2 e 5

```sql
SELECT * FROM notes
WHERE id BETWEEN 2 AND 5;
```

#### 2. Notas criadas entre duas datas

```sql
SELECT * FROM notes
WHERE created_at BETWEEN '2025-01-01' AND '2025-01-31';
```

#### 3. Valores fora de um intervalo (usando `NOT BETWEEN`)

```sql
SELECT * FROM notes
WHERE id NOT BETWEEN 2 AND 5;
```

---

👉 Resumindo:

* `BETWEEN` → inclui os limites do intervalo.
* `NOT BETWEEN` → exclui o intervalo.

---


## 11. Admin Rápido no psql (linha de comando)

```sql
-- Listar bancos
\l

-- Conectar a um banco
\c trilha_db

-- Listar tabelas
\dt

-- Estrutura de uma tabela
\d notes
```

---

