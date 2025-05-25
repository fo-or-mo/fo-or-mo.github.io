---
title: PyWebView
author: Zhang
date: 2025-05-26
category: python
layout: post
mermaid: true
---
```python
import webview
import sqlite3
from threading import Lock

def create_db():
    """初始化数据库和表结构（注意这里使用article字段）"""
    with sqlite3.connect('articles.db') as conn:
        conn.execute('''CREATE TABLE IF NOT EXISTS articles
                     (id INTEGER PRIMARY KEY AUTOINCREMENT,
                      title TEXT NOT NULL,
                      article TEXT NOT NULL)''')  # 这里保持为article
        conn.commit()

class Api:
    def __init__(self):
        self.lock = Lock()
    
    def _get_conn(self):
        return sqlite3.connect('articles.db', check_same_thread=False)
    
    def add_article(self, title, content):
        """添加新文章（注意字段名对应）"""
        if not title.strip():
            return {'error': '标题不能为空'}
        
        with self.lock, self._get_conn() as conn:
            # 这里使用article作为字段名
            conn.execute("INSERT INTO articles (title, article) VALUES (?, ?)", 
                        (title.strip(), content.strip()))
            conn.commit()
            return {'success': True}
    
    def get_articles(self):
        """获取所有文章（字段名对应）"""
        with self.lock, self._get_conn() as conn:
            cursor = conn.execute("SELECT id, title, article FROM articles ORDER BY id DESC")
            return [{'id': row[0], 'title': row[1], 'content': row[2]}
                    for row in cursor.fetchall()]
    
    def delete_article(self, article_id):
        with self.lock, self._get_conn() as conn:
            conn.execute("DELETE FROM articles WHERE id=?", (article_id,))
            conn.commit()
            return {'success': True}
    
    def update_article(self, article_id, title, content):
        if not title.strip():
            return {'error': '标题不能为空'}
        
        with self.lock, self._get_conn() as conn:
            # 这里使用article作为字段名
            conn.execute("UPDATE articles SET title=?, article=? WHERE id=?",
                        (title.strip(), content.strip(), article_id))
            conn.commit()
            return {'success': True}

# HTML部分保持原样（注意JavaScript中使用的content字段是映射关系）...

html = '''
<!DOCTYPE html>
<html>
<head>
    <title>文章管理系统</title>
    <meta charset="utf-8">
    <style>
        :root { --primary: #2196F3; --danger: #f44336; }
        body { 
            font-family: -apple-system, system-ui, sans-serif; 
            max-width: 800px; 
            margin: 0 auto; 
            padding: 20px;
        }
        .form-card {
            background: #f8f9fa;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-bottom: 30px;
        }
        .form-group { margin-bottom: 15px; }
        input, textarea {
            width: 100%;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 4px;
            box-sizing: border-box;
            font-size: 16px;
        }
        textarea { height: 150px; resize: vertical; }
        button {
            padding: 8px 20px;
            border: none;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
            transition: opacity 0.3s;
        }
        button:hover { opacity: 0.8; }
        .btn-primary {
            background: var(--primary);
            color: white;
        }
        .btn-danger {
            background: var(--danger);
            color: white;
        }
        .article-list {
            margin-top: 20px;
        }
        .article-item {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            margin-bottom: 15px;
        }
        .article-item h3 {
            margin: 0 0 10px 0;
            color: #333;
        }
        .article-item p {
            margin: 0 0 15px 0;
            color: #666;
            white-space: pre-wrap;
        }
        .button-group { display: flex; gap: 10px; }
    </style>
</head>
<body>
    <div class="form-card">
        <h2>文章编辑器</h2>
        <div class="form-group">
            <input type="text" id="title" placeholder="输入文章标题...">
        </div>
        <div class="form-group">
            <textarea id="content" placeholder="输入文章内容..."></textarea>
        </div>
        <button class="btn-primary" onclick="saveArticle()">保存文章</button>
    </div>

    <div class="article-list" id="list"></div>

    <script>
        let editingId = null;

        // 加载文章列表
        async function loadArticles() {
            const articles = await pywebview.api.get_articles()
            const listElement = document.getElementById('list')
            
            listElement.innerHTML = articles.map(article => `
                <div class="article-item" data-id="${article.id}">
                    <h3>${article.title}</h3>
                    <p>${article.content}</p>
                    <div class="button-group">
                        <button class="btn-primary" onclick="editArticle(${article.id})">编辑</button>
                        <button class="btn-danger" onclick="deleteArticle(${article.id})">删除</button>
                    </div>
                </div>
            `).join('')
        }

        // 保存/更新文章
        async function saveArticle() {
            const title = document.getElementById('title').value
            const content = document.getElementById('content').value

            let result
            try {
                if (editingId) {
                    result = await pywebview.api.update_article(editingId, title, content)
                } else {
                    result = await pywebview.api.add_article(title, content)
                }

                if (result.error) {
                    alert(result.error)
                    return
                }

                // 重置表单
                editingId = null
                document.getElementById('title').value = ''
                document.getElementById('content').value = ''
                await loadArticles()
            } catch (error) {
                console.error('操作失败:', error)
                alert('操作失败，请检查控制台')
            }
        }

        // 编辑文章
        async function editArticle(id) {
            const articles = await pywebview.api.get_articles()
            const article = articles.find(a => a.id === id)
            
            document.getElementById('title').value = article.title
            document.getElementById('content').value = article.content
            editingId = id
            
            window.scrollTo({
                top: 0,
                behavior: 'smooth'
            })
        }

        // 删除文章
        async function deleteArticle(id) {
            if (confirm('确定要删除这篇文章吗？此操作不可撤销！')) {
                try {
                    await pywebview.api.delete_article(id)
                    await loadArticles()
                } catch (error) {
                    console.error('删除失败:', error)
                    alert('删除失败，请检查控制台')
                }
            }
        }

        // 初始化加载
        loadArticles()
    </script>
</body>
</html>
'''

if __name__ == '__main__':
    create_db()  # 初始化数据库
    api = Api()
    window = webview.create_window(
        title='文章管理系统',
        html=html,
        js_api=api,
        width=900,
        height=700
    )
    webview.start()
```
